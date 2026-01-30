# Kafka Consumer Deployment to Omega EKS: Complete Guide

This document details the deployment of the `kafka_consumer` application to Yahoo's Omega EKS (Kubernetes) platform, including all issues encountered, their root causes, and solutions.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Issues and Resolutions](#issues-and-resolutions)
   - [Issue 1: Config File Not Found](#issue-1-config-file-not-found)
   - [Issue 2: CKMS Key Retrieval Failing](#issue-2-ckms-key-retrieval-failing)
   - [Issue 3: Health Check Probe Failures](#issue-3-health-check-probe-failures)
   - [Issue 4: Kafka "Network is Unreachable"](#issue-4-kafka-network-is-unreachable)
4. [Key Concepts Explained](#key-concepts-explained)
5. [Final Configuration](#final-configuration)
6. [Lessons Learned](#lessons-learned)

---

## Overview

### What We Were Deploying

A Python-based Kafka consumer application that:
- Connects to Amazon MSK (Managed Streaming for Kafka) via YCPI (Yahoo Cloud Private Infrastructure)
- Processes BigPanda incident alerts from Kafka topics
- Executes automated remediation actions based on configurable incident conditions
- Uses SCRAM-SHA-512 authentication with credentials from CKMS

### The Challenge

Moving from traditional EC2/on-prem deployment to Omega EKS (Kubernetes) required:
- Adapting to Omega's container lifecycle and health check mechanisms
- Configuring Athenz identity for CKMS secret retrieval
- Solving network connectivity differences between EC2 and EKS environments

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              OMEGA EKS CLUSTER                                  │
│                     (omega-aws.centraltech-nonprod1.use1)                       │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    POD: kafka-consumer--development-use1                 │   │
│  │                                                                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │   │
│  │  │ kafka-       │  │ sia          │  │ istio-proxy  │                   │   │
│  │  │ consumer     │  │ (Athenz)     │  │ (sidecar)    │                   │   │
│  │  │ container    │  │              │  │              │                   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                   │   │
│  │         │                 │                                              │   │
│  │         │    /var/run/athenz/service.cert.pem                           │   │
│  │         │    /var/run/athenz/service.key.pem                            │   │
│  │         │                                                                │   │
│  └─────────┼────────────────────────────────────────────────────────────────┘   │
│            │                                                                     │
│            │ NAT Gateway (52.44.114.41, 44.217.139.107, 44.221.248.67)          │
└────────────┼─────────────────────────────────────────────────────────────────────┘
             │
             │ TCP:443 (IPv4 only - IPv6 not routed)
             ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              YCPI VIP LAYER                                     │
│                    (e1.ycpi.vip.dca.yahoo.com:443)                              │
│                                                                                 │
│  DNS: broker1-public-bpimskprod-c21.yahooinc.com                               │
│    → IPv4: 69.147.92.11, 69.147.92.12  ✅ Works                                │
│    → IPv6: 2001:4998:14:800::1000      ❌ No route from Omega                  │
└─────────────────────────────────────────────────────────────────────────────────┘
             │
             │ "Partial Blind Tunnel" routing
             ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           AMAZON MSK CLUSTER                                    │
│                         (bpi-msk-prod / bpimskprod)                             │
│                                                                                 │
│  Broker 1: broker1-public-bpimskprod-c21.yahooinc.com:443                      │
│  Broker 2: broker2-public-bpimskprod-c21.yahooinc.com:443                      │
│  Broker 3: broker3-public-bpimskprod-c21.yahooinc.com:443                      │
│                                                                                 │
│  Authentication: SCRAM-SHA-512 (username/password from CKMS)                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Issues and Resolutions

### Issue 1: Config File Not Found

**Error:**
```
ERROR: Config file not found at /tmp/app-configmap/config.yaml
```

**Root Cause:**
Omega expects applications to use its native config injection mechanism via `custom.appSrcDir` and `custom.appTargetDir` in `omega.yaml`. We initially tried various approaches including baking config into the Docker image and using `configMounts`.

**Solution:**
Configure Omega's native config injection in `omega.yaml`:

```yaml
custom:
  appLogsDirs:
    - /app/logs
    - /tmp/logs
  appSrcDir: deploy_target/omega/config    # Source in repo
  appTargetDir: /app/config                 # Mount point in container
```

And update `app_start` to use the correct path:
```bash
DEFAULT_CONFIG_FILE="/app/config/config.yaml"
CONFIG_FILE="${CONFIG_FILE:-$DEFAULT_CONFIG_FILE}"
```

**Concept: Omega Config Injection**

Omega automatically takes files from `appSrcDir` in your repository and mounts them at `appTargetDir` in the container. This happens during the Screwdriver build process via `omegagen`, which creates ConfigMaps from your config files.

---

### Issue 2: CKMS Key Retrieval Failing

**Error:**
```
[ERROR] Failed to get CKMS key!
Exception: Failed to get CKMS key: group=bigpanda.keys.sre, key=bigpanda.prod.api.sre
```

**Debug Output:**
```
[ckms-remotecli debug] [ckms-error] CKMS request failed (3 retries): Failed to perform HTTP request: 
Post "https://dek.ckms.ouroath.com:4443/rksd/v1/access": read tcp 10.232.x.x:443: connection reset by peer
```

**Root Cause:**
Two issues combined:

1. **Athenz service not properly configured for Omega/Kubernetes**
   - Missing "Kubernetes (Omega) launches instances for the service" flag
   - Missing "Allow ZTS as an identity provider for the service" flag

2. **CKMS client not explicitly set to AWS environment**
   - The `ouroath.ckms` library needs `env = 'aws'` to be explicitly set

**Solution:**

1. In Athenz UI, enable for the service (`sre.kafka-consumer-nonprod-k8s.kafka-consumer-development`):
   - ✅ Kubernetes (Omega) launches instances for the service
   - ✅ Allow ZTS as an identity provider for the service

2. In `secret_manager.py`, explicitly set the CKMS environment:
```python
ckms_client = YSecureOathCKMS()
ckms_client.env = 'aws'  # Explicitly set AWS environment
ckms_client.cert_filename = x509_cert
ckms_client.key_filename = x509_key
```

3. In `app_start`, export the environment variable:
```bash
export CKMS_ENV="aws"
```

**Concept: Athenz Identity in Omega**

Omega uses a sidecar container called `sia` (Service Identity Agent) to automatically provision Athenz certificates for your application. These certificates are mounted at:
- `/var/run/athenz/service.cert.pem` - X.509 certificate
- `/var/run/athenz/service.key.pem` - Private key

The certificate's CN (Common Name) follows the format:
```
CN=<athenz-domain>.<service-name>
Example: sre.kafka-consumer-nonprod-k8s.kafka-consumer-development
```

This identity is used for:
- Authenticating to CKMS for secret retrieval
- mTLS with other services
- Authorization decisions in Athenz policies

---

### Issue 3: Health Check Probe Failures

**Error:**
```
PODCTL: PROBE=readiness ERROR: non-200 response. Your app is still starting up, OR crashing, 
OR missing a required endpoint.
Command was /usr/bin/curl --silent --output /dev/null --connect-timeout 3 --max-time 5 
--retry 3 --write-out %{http_code} --insecure localhost:4080/status.html
```

**Root Cause:**
By default, Omega expects applications to expose an HTTP health check endpoint at `localhost:4080/status.html`. Our Kafka consumer is a background process that doesn't serve HTTP traffic - it just consumes from Kafka and processes messages.

**Solution:**
Configure custom health checks in `omega.yaml`:

```yaml
# Use shell scripts for health checks (no HTTP server in this app)
probe: custom
```

Create health check scripts that verify the consumer process is running:

**`deploy_target/omega/scripts/app_liveness`:**
```bash
#!/bin/bash
PIDFILE="/var/run/kafka_consumer/consumer.pid"

if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE")
    if ps -p "$PID" > /dev/null; then
        echo "kafka_consumer process (PID: $PID) is running."
        exit 0
    fi
fi
exit 1
```

**`deploy_target/omega/scripts/app_readiness`:**
```bash
#!/bin/bash
PIDFILE="/var/run/kafka_consumer/consumer.pid"

if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE")
    if ps -p "$PID" > /dev/null; then
        echo "kafka_consumer process (PID: $PID) is running and ready."
        exit 0
    fi
fi
exit 1
```

Update `app_start` to write a PID file:
```bash
/opt/y/1.0/bin/kafka_consumer --config "$CONFIG_FILE" --log-level "$LOG_LEVEL" &
echo $! > /var/run/kafka_consumer/consumer.pid
wait
```

**Concept: Kubernetes Health Probes**

Kubernetes uses three types of probes:

| Probe | Purpose | On Failure |
|-------|---------|------------|
| **Liveness** | Is the app alive? | Restart container |
| **Readiness** | Can it receive traffic? | Remove from service endpoints |
| **Startup** | Has it finished starting? | Keep checking, then hand off to liveness |

For HTTP apps, probes typically hit an endpoint. For non-HTTP apps like our consumer, `probe: custom` tells Omega to use scripts instead:
- `/scripts/app_liveness` - Called for liveness checks
- `/scripts/app_readiness` - Called for readiness checks

The scripts must exit with code 0 for healthy, non-zero for unhealthy.

---

### Issue 4: Kafka "Network is Unreachable"

**Error:**
```
%3|1769755513.773|FAIL|kafka-consumer#consumer-1| [thrd:GroupCoordinator]: 
broker1-public-bpimskprod-c21.yahooinc.com:443: Failed to connect to broker at
[e1.ycpi.vip.dca.yahoo.com]:443: Network is unreachable (after 7ms in state CONNECT)
```

**Initial Hypothesis (Wrong):**
We initially suspected YCPI/MSK network allow-lists were blocking traffic from Omega's NAT IP addresses.

**Debugging Process:**

1. **Tested raw TCP connectivity from inside the pod:**
```bash
# IPv4 - WORKS!
nc -zv broker1-public-bpimskprod-c21.yahooinc.com 443
Ncat: Connected to 69.147.92.11:443.

# IPv6 - FAILS!
nc -6 -zv broker1-public-bpimskprod-c21.yahooinc.com 443
Ncat: Connection to 2001:4998:14:800::1001 failed: Network is unreachable.
```

2. **Checked DNS records:**
```bash
getent ahosts broker1-public-bpimskprod-c21.yahooinc.com
69.147.92.11    STREAM edge.gycpi.b.yahoodns.net   # IPv4
69.147.92.12    STREAM                              # IPv4
2001:4998:14:800::1001 STREAM                       # IPv6
2001:4998:14:800::1000 STREAM                       # IPv6
```

**Actual Root Cause:**
The broker DNS returns both IPv4 (A) and IPv6 (AAAA) records. librdkafka's default behavior (`broker.address.family: any`) follows the OS address selection, which typically prefers IPv6. The Omega EKS cluster does not have IPv6 routing enabled, so:

1. librdkafka resolves broker hostname → Gets both IPv4 and IPv6 addresses
2. Tries IPv6 first (OS preference) → Kernel immediately returns ENETUNREACH (no route)
3. librdkafka treats this as a broker failure, not just an address failure
4. Connection fails without trying IPv4

**Solution:**
Force librdkafka to use IPv4 only in `automation_consumer.py`:

```python
config: Dict[str, Any] = {
    'auto.offset.reset': 'latest',
    'bootstrap.servers': f"{','.join(self.kafka_brokers)}",
    'broker.address.family': 'v4',  # Force IPv4 - Omega EKS lacks IPv6 routing
    'client.id': socket.gethostname(),
    'enable.auto.commit': True,
    'group.id': self.kafka_consumer_group_id,
    'security.protocol': 'SASL_SSL',
}
```

**Concept: IPv4 vs IPv6 in Kubernetes/Cloud**

Many cloud Kubernetes clusters, including Omega EKS, are IPv4-only. When a DNS hostname has both A (IPv4) and AAAA (IPv6) records:

- The OS typically prefers IPv6 (per RFC 6724)
- If there's no IPv6 route, connection attempts fail immediately with ENETUNREACH
- Some libraries don't gracefully fall back to IPv4

The `broker.address.family` librdkafka setting controls this:
- `any` (default) - Use whatever getaddrinfo() returns (may try IPv6 first)
- `v4` - Only request IPv4 addresses
- `v6` - Only request IPv6 addresses

**Concept: YCPI (Yahoo Cloud Private Infrastructure)**

YCPI is Yahoo's internal network routing layer that fronts services like MSK:

```
On-prem/Cloud Client → YCPI VIP → Backend Service (MSK)
```

YCPI creates DNS records like `broker1-public-bpimskprod-c21.yahooinc.com` that route to VIPs like `e1.ycpi.vip.dca.yahoo.com`. These VIPs use "partial blind tunnels" to proxy traffic to the actual MSK brokers.

YCPI is publicly accessible - it doesn't restrict by source IP. Our issue was IPv6, not YCPI blocking.

---

## Key Concepts Explained

### Omega EKS

Yahoo's managed Kubernetes platform running on AWS EKS. Key features:
- Automatic Athenz identity provisioning via SIA sidecar
- Istio service mesh for traffic management
- Integration with Screwdriver for CI/CD
- Network policies managed via Terraform/MSD

### Athenz

Yahoo's open-source identity and access management system:
- Provides X.509 certificates for service identity
- Controls access to resources via policies
- Used for CKMS authentication, service-to-service auth, etc.

### CKMS (Centralized Key Management System)

Yahoo's secrets management system:
- Stores API keys, passwords, certificates
- Access controlled by Athenz policies
- Secrets organized by key groups

### MSD (Micro-Segmentation for Distributed Systems)

Yahoo's network segmentation framework:
- Controls which services can communicate
- Managed via Terraform modules
- Creates Kubernetes NetworkPolicies

### librdkafka

The C library underlying most Kafka clients (including confluent-kafka-python):
- Handles Kafka protocol, connection management, consumer groups
- Many tunable settings via config dict
- `broker.address.family` controls IPv4/IPv6 behavior

---

## Final Configuration

### omega.yaml (Key Sections)

```yaml
template: generic-app:stable
appName: kafka-consumer
athensDomain: sre.kafka-consumer-nonprod-k8s

base: docker.ouroath.com:4443/yahoo-sre/yahoo.kafka_consumer:latest

# Allow time for Athenz cert provisioning
progressDeadlineSeconds: 1200

# Enable Athenz identity
requireTLSCerts: true
identityv3: true

# Custom health checks (no HTTP server)
probe: custom

# Config injection
custom:
  appLogsDirs:
    - /app/logs
  appSrcDir: deploy_target/omega/config
  appTargetDir: /app/config

jobs:
  component:
    environment: development
    colo: use1
    cluster: omega-aws
    clusterEnvironment: centraltech-nonprod1
    autoscale:
      minReplicas: 1
      maxReplicas: 1
```

### Dockerfile (Key Sections)

```dockerfile
FROM docker.ouroath.com:4443/yahoo/alma9/ubi:latest

# ... Python setup ...

# Copy health check and lifecycle scripts
COPY deploy_target/omega/scripts/ /scripts/
RUN chmod +x /scripts/*

ENTRYPOINT ["/scripts/app_start"]
```

### Network Policy (Terraform)

```hcl
module "tf-omega-app-network-policy" {
  source  = "edge.artifactory.ouroath.com:4443/omega-tf-modules__yahoo-cloud/tf-omega-app-network-policy/athenz"
  version = "~>0.0.7"

  athenz_domain   = local.domain
  omega_app_name  = "kafka-consumer"
  omega_app_ports = [4080]
  
  # Default allows egress to 443, 4443, 80, 4080, 53, etc.
  
  cloud_provider = "omega-aws"
  cloud_clusters = ["centraltech-nonprod1"]
}
```

---

## Lessons Learned

### 1. Debug Layer by Layer

When facing connectivity issues, test each layer independently:
- DNS resolution: `getent ahosts hostname`
- TCP connectivity: `nc -zv hostname port`
- IPv4 vs IPv6: `nc -4 -zv` vs `nc -6 -zv`
- TLS handshake: `openssl s_client -connect hostname:port`

### 2. Check IPv6 Early

Modern DNS often returns both A and AAAA records. If your environment doesn't support IPv6, force IPv4 explicitly rather than relying on fallback behavior.

### 3. Athenz Requires Explicit Enablement

For services running in Omega/Kubernetes:
- Enable "Kubernetes (Omega) launches instances for the service"
- Enable "Allow ZTS as an identity provider for the service"
- Add service principals to CKMS access groups

### 4. Custom Health Checks for Non-HTTP Apps

Use `probe: custom` and script-based checks for applications that:
- Don't serve HTTP traffic
- Are background workers/consumers
- Have complex readiness criteria

### 5. Document the Journey

Complex deployments involve many moving parts. Document issues and solutions as you go - it helps future deployments and other team members.

---

## Timeline Summary

| Date | Issue | Resolution |
|------|-------|------------|
| Day 1 | omega.yaml validation errors | Fixed `replicas`, removed invalid `configMounts` |
| Day 1 | Docker image path wrong | Fixed to `yahoo-sre/yahoo.kafka_consumer` |
| Day 1 | PVC not found | Created `kafka-consumer-history-dev` PVC |
| Day 1-2 | Config file not found | Used `custom.appSrcDir` for config injection |
| Day 2 | CKMS key retrieval failing | Enabled Athenz ZTS/Omega flags, set `env='aws'` |
| Day 2-3 | Health check failures | Added `probe: custom` with PID-based scripts |
| Day 3 | "Network is unreachable" | **Root cause: IPv6**. Fixed with `broker.address.family: v4` |

---

## Verification Commands

```bash
# Check pod status
kubectl get pods -n sre-kafka--consumer--nonprod--k8s

# View consumer logs
kubectl -n sre-kafka--consumer--nonprod--k8s logs -f <pod-name> -c kafka-consumer

# Check consumer process inside pod
kubectl -n sre-kafka--consumer--nonprod--k8s exec <pod-name> -c kafka-consumer -- \
  ps aux | grep kafka_consumer

# Test broker connectivity from pod
kubectl -n sre-kafka--consumer--nonprod--k8s exec <pod-name> -c kafka-consumer -- \
  nc -zv broker1-public-bpimskprod-c21.yahooinc.com 443
```

---

*Last Updated: January 30, 2026*
*Author: SRE Team*
