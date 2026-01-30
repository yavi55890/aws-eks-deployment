# Omega EKS Deployment: Complete Troubleshooting Guide & Action Plan

> **Document Purpose:** This document chronicles the complete journey of deploying the `kafka-consumer` application to Omega EKS, including all issues encountered, resolutions applied, and the current blocking issue requiring external team intervention.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Deployment Timeline & Issues Resolved](#deployment-timeline--issues-resolved)
3. [Current Blocker: MSK/YCPI Network Connectivity](#current-blocker-mskypi-network-connectivity)
4. [Understanding the Architecture](#understanding-the-architecture)
5. [Detailed Action Plan](#detailed-action-plan)
6. [Commands Reference](#commands-reference)
7. [Lessons Learned](#lessons-learned)
8. [Appendix: Complete Configuration Reference](#appendix-complete-configuration-reference)

---

## Executive Summary

### Current Status

| Component | Status | Notes |
|-----------|--------|-------|
| Docker Image Build | ✅ Complete | Image pushed to `docker.ouroath.com:4443/yahoo-sre/yahoo.kafka_consumer:latest` |
| Omega Deployment | ✅ Deployed | Pods running in `sre-kafka--consumer--nonprod--k8s` namespace |
| Configuration Injection | ✅ Working | Config baked into image via `custom.appSrcDir` |
| Athenz Identity | ✅ Provisioned | Certs at `/var/run/athenz/` |
| CKMS Secret Retrieval | ✅ Working | All 4 keys retrieved successfully |
| Health Checks | ✅ Configured | Custom liveness/readiness probes |
| **Kafka Connectivity** | ❌ **BLOCKED** | Network unreachable to YCPI VIPs |

### What's Blocking Us

The Kafka consumer successfully starts but **cannot connect to MSK brokers**. The error:

```
Failed to connect to broker at [e1.ycpi.vip.dca.yahoo.com]:443: Network is unreachable
```

**Root Cause:** The Omega EKS cluster's egress IP addresses are **not in the MSK/YCPI network allow-lists**. This is an external infrastructure issue that requires the MSK/YCPI team to resolve.

---

## Deployment Timeline & Issues Resolved

### Issue 1: omega.yaml Validation Errors

**Error:**
```
omegagen validation error: (root): Additional property configMounts is not allowed
omegagen validation error: (root): Additional property replicas is not allowed
```

**Resolution:**
- `replicas` moved into `autoscale` section
- `configMounts` removed from root level (later addressed via config baking)

---

### Issue 2: Docker Image Not Found

**Error:**
```
docker.ouroath.com:4443/yahoo/kafka_consumer:latest: not found
```

**Resolution:**
Changed `omega.yaml` to use correct image path:

```yaml
# Before (incorrect)
base: docker.ouroath.com:4443/yahoo/kafka_consumer

# After (correct)
appImage: docker.ouroath.com:4443/yahoo-sre/yahoo.kafka_consumer:latest
```

---

### Issue 3: PVC Not Found

**Error:**
```
persistentvolumeclaim "kafka-consumer-history-dev" not found
```

**Resolution:**
Applied PVC in the correct Kubernetes context:

```bash
# Switch to correct context
kubectl config use-context omega-aws.centraltech-nonprod1-kdnc.use1

# Apply PVC
kubectl apply -f deploy_target/omega/pvc-nonprod.yaml
```

---

### Issue 4: Config File Not Found

**Error:**
```
ERROR: Config file not found at /app/config/config.yaml
```

**Resolution:**
Used `custom.appSrcDir` and `custom.appTargetDir` in `omega.yaml` to bake config into image:

```yaml
custom:
  appSrcDir: deploy_target/omega/config
  appTargetDir: /app/config
```

And simplified `app_configure` script to only create runtime directories.

---

### Issue 5: CKMS Key Retrieval Failed

**Error:**
```
[ERROR] Failed to get CKMS key: group=bigpanda.keys.sre, key=bigpanda.prod.api.sre
```

**Debug Output:**
```
connection reset by peer... Failed to perform HTTP request to dek.ckms.ouroath.com:4443
```

**Root Cause:**
Two Athenz settings were not enabled for the service.

**Resolution:**
1. Enabled in Athenz UI for `sre.kafka-consumer-nonprod-k8s`:
   - ☑️ **Kubernetes (Omega) launches instances for the service**
   - ☑️ **Allow ZTS as an identity provider for the service**

2. Added explicit environment setting in `secret_manager.py`:
   ```python
   ckms_client = YSecureOathCKMS()
   ckms_client.env = 'aws'  # Critical for AWS CKMS endpoint
   ```

3. Added `CKMS_ENV=aws` in `app_start` script as belt-and-suspenders.

---

### Issue 6: Readiness Probe Failing

**Error:**
```
PODCTL: PROBE=readiness ERROR: non-200 response
Command was /usr/bin/curl localhost:4080/status.html
```

**Root Cause:**
`omega.yaml` had `probe: custom` but no custom probe scripts existed.

**Resolution:**
Created `app_liveness` and `app_readiness` scripts that check for PID file:

```bash
#!/bin/bash
# app_liveness / app_readiness
PIDFILE="/var/run/kafka_consumer/consumer.pid"
if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE")
    if ps -p "$PID" > /dev/null; then
        exit 0
    fi
fi
exit 1
```

---

### Issue 7: Kafka Network Unreachable (CURRENT BLOCKER)

**Error:**
```
%3|...|FAIL|... Failed to connect to broker at [e1.ycpi.vip.dca.yahoo.com]:443: Network is unreachable
```

**Status:** ⏳ **Waiting for MSK/YCPI team to update network allow-lists**

See [Current Blocker](#current-blocker-mskypi-network-connectivity) section for details.

---

## Current Blocker: MSK/YCPI Network Connectivity

### Problem Summary

The Kafka consumer application:
- ✅ Starts successfully
- ✅ Retrieves all CKMS secrets
- ✅ Resolves DNS for MSK brokers
- ❌ **Cannot establish TCP connection to YCPI VIPs**

### Why This Happens

```
┌─────────────────────────────────────────────────────────────┐
│              Omega EKS Cluster                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  kafka-consumer Pod                                  │   │
│   │  [attempts connection to broker:443]                 │   │
│   └──────────────────────────────────────────────────────┘   │
│                            │                                 │
│                            ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  K8s NetworkPolicy                                   │   │
│   │  ✅ Allows egress to port 443                        │   │
│   └──────────────────────────────────────────────────────┘   │
│                            │                                 │
│                            ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  AWS NAT Gateway                                     │   │
│   │  Source IP: x.x.x.x (NOT in allow-list)              │   │
│   └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    YCPI Layer                                │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  MSD Transport Policy / WAKL                         │   │
│   │  ❌ Omega NAT IPs NOT in allowed prefix list         │   │
│   │                                                      │   │
│   │  RESULT: Connection dropped / unreachable            │   │
│   └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### What You Control vs What You Don't

| Layer | Who Controls | Your Responsibility | Status |
|-------|--------------|---------------------|--------|
| Application Code | You | Configure brokers, auth | ✅ Done |
| omega.yaml | You | Deployment config | ✅ Done |
| Terraform NetworkPolicy | You | K8s egress rules | ✅ Done |
| YCPI/MSD Allow-lists | **MSK/YCPI Team** | N/A - External | ❌ Blocked |
| MSK Security Groups | **MSK Team** | N/A - External | ❌ Blocked |

### Key Insight

**There is NO `omega.yaml` change that can fix this.** The traffic is being dropped at the YCPI/MSD network layer, which is external infrastructure managed by a different team.

---

## Understanding the Architecture

### Component Breakdown

#### 1. Your Application Stack

| File | Purpose |
|------|---------|
| `deploy_target/omega/omega.yaml` | Kubernetes deployment manifest for Omega |
| `deploy_target/omega/config/config.yaml` | Application config (brokers, secrets, workflows) |
| `deploy_target/omega/scripts/app_start` | Container startup script |
| `deploy_target/omega/scripts/app_configure` | Image build-time configuration |
| `deploy_target/omega/scripts/app_liveness` | Liveness probe (is app running?) |
| `deploy_target/omega/scripts/app_readiness` | Readiness probe (is app ready?) |
| `src/yahoo/kafka_consumer/secret_manager.py` | CKMS integration for secrets |

#### 2. External Infrastructure (Not Controlled by You)

| Component | Owner | Purpose |
|-----------|-------|---------|
| YCPI VIPs | YCPI Team | Network load balancing, routing |
| MSD Transport Policies | YCPI/Athenz Team | IP-based network allow-lists |
| MSK Security Groups | MSK Team | AWS-level firewall for MSK |
| WAKL | YCPI Team | Web Application Key List (IP allow-list) |

#### 3. How MSK Connectivity Works

```
Your Pod → K8s NetworkPolicy → NAT Gateway → YCPI VIP → MSK Broker
                                              ↑
                                    This is where packets
                                    are being dropped
```

The broker hostnames (`broker1-public-bpimskprod-c21.yahooinc.com`) resolve to YCPI VIPs (`e1.ycpi.vip.dca.yahoo.com`). YCPI then routes to MSK, but only if the source IP is in the allow-list.

#### 4. YCPI Deep Dive: How the MSK Proxy Works

YCPI (Yahoo Cloud Platform Infrastructure) acts as a **network gateway/proxy** between consumers and MSK. Here's the complete picture:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           YOUR APPLICATION                                    │
│                                                                              │
│  config.yaml:                                                                │
│    brokers:                                                                  │
│      - broker1-public-bpimskprod-c21.yahooinc.com:443  ◄── These are NOT    │
│      - broker2-public-bpimskprod-c21.yahooinc.com:443      the real MSK     │
│      - broker3-public-bpimskprod-c21.yahooinc.com:443      brokers!         │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ DNS Lookup
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              DNS RESOLUTION                                   │
│                                                                              │
│  broker1-public-bpimskprod-c21.yahooinc.com                                  │
│         │                                                                    │
│         └──► Resolves to: e1.ycpi.vip.dca.yahoo.com  ◄── YCPI VIP           │
│                                                                              │
│  These DNS records were created BY YCPI specifically for this MSK cluster   │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ TCP Connection to port 443
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         YCPI LAYER (Load Balancer/Proxy)                     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                        Edgeman Routing Rules                           │  │
│  │                        (SRE_US property)                               │  │
│  │                                                                        │  │
│  │  Rule: "Partial Blind Tunnel" (P: prefix)                              │  │
│  │                                                                        │  │
│  │  IF request to: broker1-public-bpimskprod-c21.yahooinc.com:443        │  │
│  │  THEN tunnel to: [actual MSK broker endpoint]:9196                    │  │
│  │                                                                        │  │
│  │  Access Control:                                                       │  │
│  │  - MSD policies check source IP                                        │  │
│  │  - WAKL contains allowed IP prefixes                                   │  │
│  │  - YOUR NAT IPs need to be in this list! ◄── This is what YCPI-6483   │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Internal routing (if allowed)
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         AMAZON MSK CLUSTER                                    │
│                         (bpi-msk-prod)                                        │
│                                                                              │
│  Actual Brokers (hidden behind YCPI):                                        │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                │
│  │   Broker 1      │ │   Broker 2      │ │   Broker 3      │                │
│  │   Port: 9196    │ │   Port: 9196    │ │   Port: 9196    │                │
│  │   (SCRAM/TLS)   │ │   (SCRAM/TLS)   │ │   (SCRAM/TLS)   │                │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                │
│                                                                              │
│  Security Groups:                                                            │
│  - Reference AWS Prefix Lists (msk-access-prefix-list-{0,1})                 │
│  - Allow ingress on port 9196 from YCPI IP ranges only                      │
│  - Updated hourly by WAKL Lambda                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

##### Why YCPI Exists

| Function | Explanation |
|----------|-------------|
| **Unified Access Point** | On-prem and cloud consumers use the same DNS names |
| **Port Translation** | You connect on `443`, YCPI forwards to `9196` (MSK's actual port) |
| **Access Control** | IP-based allow-listing via MSD/WAKL |
| **Load Balancing** | Distributes traffic across brokers |
| **Network Bridging** | Routes traffic between Yahoo on-prem networks and AWS VPCs |

##### The DNS Records Are Synthetic

The broker hostnames you're using:
```
broker1-public-bpimskprod-c21.yahooinc.com
broker2-public-bpimskprod-c21.yahooinc.com
broker3-public-bpimskprod-c21.yahooinc.com
```

These are **NOT** the actual MSK broker endpoints. They're DNS records created by YCPI that:

1. **Resolve to YCPI VIPs** (like `e1.ycpi.vip.dca.yahoo.com`)
2. **Get routed through YCPI** using rules defined in Edgeman
3. **Eventually reach the real MSK brokers** on port 9196

The real MSK broker endpoints look more like:
```
b-1.bpimskprod.xxxxx.c21.kafka.us-east-1.amazonaws.com:9196
```

But you never see or use those directly.

##### The Access Control Chain

```
Your Request → YCPI checks your source IP → Is it in the prefix list?
                                                    │
                    ┌───────────────────────────────┴───────────────────────────────┐
                    │                                                               │
                    ▼                                                               ▼
              YES: Forward to MSK                                      NO: Drop/Reject
              (via blind tunnel)                                       "Network unreachable"
```

##### How the Prefix Lists Get Updated (WAKL)

```
┌─────────────────────┐     hourly      ┌─────────────────────┐
│   WAKL Lambda       │ ───────────────►│  AWS Prefix Lists   │
│   (runs every hour) │                 │  (msk-access-*)     │
└─────────────────────┘                 └─────────────────────┘
         │                                        │
         │ Queries                                │ Referenced by
         ▼                                        ▼
┌─────────────────────┐                 ┌─────────────────────┐
│   MSD Policies      │                 │  Security Groups    │
│   (Athenz)          │                 │  (MSK cluster)      │
└─────────────────────┘                 └─────────────────────┘
```

The WAKL (Wide Area Key List) Lambda fetches YCPI IP ranges from MSD and updates AWS Prefix Lists hourly. Security Groups reference these prefix lists.

##### MSD Transport Policy Example

This is what the MSK team configured to allow YCPI traffic:

```hcl
transport_policies = [
  {
    identifier = "bpi-msk-access-{env}-allow-from-ycpi"
    direction  = "ingress"
    remote_services = [
      "ycpi.ycpi-networks-access"
    ]
    destination_ports = ["9196"]
    source_ports      = ["1024-65535"]
    protocol          = "TCP"
    conditions = [
      {
        hosts             = "*"
        scope             = ["aws"]
        enforcement_state = "enforce"
      }
    ]
  }
]
```

##### Port Translation Summary

| Layer | What It Is | Port |
|-------|-----------|------|
| **Your config** | `broker1-public-bpimskprod-c21.yahooinc.com` | 443 |
| **DNS resolves to** | YCPI VIP (`e1.ycpi.vip.dca.yahoo.com`) | 443 |
| **YCPI tunnels to** | Real MSK broker | 9196 |
| **Real MSK endpoint** | `b-1.bpimskprod.xxx.kafka.us-east-1.amazonaws.com` | 9196 |

##### Related Jira Tickets

- **[YCPI-6285](https://ouryahoo.atlassian.net/browse/YCPI-6285)**: Original onboarding to proxy requests from on-prem to Amazon MSK
- **[YCPI-6483](https://ouryahoo.atlassian.net/browse/YCPI-6483)**: Request to add Omega EKS NAT IPs to allow-lists (current ticket)

#### 5. Omega Cluster NAT IPs

These are the NAT Gateway IPs that need to be allow-listed for MSK access:

**Non-Production Cluster:** `omega-aws.centraltech-nonprod1.use1`
| IP Address |
|------------|
| 52.44.114.41 |
| 44.217.139.107 |
| 44.221.248.67 |

**Production Cluster:** `omega-aws.centraltech-prod1.use1`
| IP Address |
|------------|
| 3.224.78.221 |
| 18.235.98.75 |
| 3.214.29.144 |

#### 6. Terraform Network Policies (Already Configured)

```hcl
# terraform/athenz/sre.kafka-consumer-nonprod-k8s/network-policies.tf
module "tf-omega-app-network-policy" {
  source  = "edge.artifactory.ouroath.com:4443/omega-tf-modules__yahoo-cloud/tf-omega-app-network-policy/athenz"
  version = "~>0.0.7"

  athenz_domain   = local.domain             # sre.kafka-consumer-nonprod-k8s
  omega_app_name  = "kafka-consumer"
  omega_app_ports = [4080]

  # Default egress allows: 443, 4443, 80, 4080, 53
  cloud_provider = local.cloud_provider
  cloud_clusters = local.cloud_clusters
}
```

This creates the Kubernetes NetworkPolicy, which **already allows egress to port 443**:

```bash
$ kubectl describe networkpolicy sre.kafka-consumer-nonprod-k8s.kafka-consumer-development
...
Allowing egress traffic:
  To Port Range: 443-443/TCP   ← Kafka brokers use this
  To Port Range: 4443-4443/TCP ← CKMS uses this
  ...
```

---

## Detailed Action Plan

### ✅ Completed Actions

| # | Action | Status | Notes |
|---|--------|--------|-------|
| 1 | Create omega.yaml with correct structure | ✅ Done | Fixed validation errors |
| 2 | Fix Docker image path | ✅ Done | `yahoo-sre/yahoo.kafka_consumer` |
| 3 | Create PVC for action history | ✅ Done | Applied to correct context |
| 4 | Configure config injection | ✅ Done | Using `appSrcDir`/`appTargetDir` |
| 5 | Enable Athenz ZTS/Omega settings | ✅ Done | In Athenz UI |
| 6 | Fix CKMS client environment | ✅ Done | Added `ckms_client.env = 'aws'` |
| 7 | Create custom health check scripts | ✅ Done | `app_liveness`, `app_readiness` |
| 8 | Remove debug logging | ✅ Done | Cleaned up `secret_manager.py` |

### ⏳ Pending Actions (External Team Required)

| # | Action | Owner | Priority | Tracking |
|---|--------|-------|----------|----------|
| 1 | **Add Omega NAT IPs to YCPI/MSD allow-lists** | MSK/YCPI Team | **CRITICAL** | Create Jira ticket |
| 2 | **Add Omega NAT IPs to MSK Security Groups** | MSK Team | **CRITICAL** | Same ticket |
| 3 | Verify recommended path (YCPI vs PrivateLink) | MSK Team | High | Ask in ticket |

### 📋 Request Template for MSK/YCPI Team

Copy this into a Jira ticket or Slack message:

```
Subject: Request to add Omega EKS cluster to MSK/YCPI allow-lists for kafka-consumer

Hi MSK/YCPI Team,

We are deploying a Kafka consumer application to Omega EKS and encountering
"Network is unreachable" errors when connecting to MSK brokers.

=== Application Details ===
- Application: kafka-consumer
- Omega Cluster: omega-aws.centraltech-nonprod1.use1
- Namespace: sre-kafka--consumer--nonprod--k8s
- Athenz Domain: sre.kafka-consumer-nonprod-k8s
- Athenz Service: kafka-consumer-development
- Full Principal: sre.kafka-consumer-nonprod-k8s.kafka-consumer-development

=== MSK Cluster Details ===
- Cluster: bpi-msk-prod
- Authentication: SASL/SCRAM
- Port: 9198 / 443 (via YCPI)
- Brokers:
  - b-1-public.alerts.kafka.us-east-1.amazonaws.com:9198
  - b-2-public.alerts.kafka.us-east-1.amazonaws.com:9198
  - b-3-public.alerts.kafka.us-east-1.amazonaws.com:9198

=== Observed Error ===
Failed to connect to broker at [e1.ycpi.vip.dca.yahoo.com]:443: Network is unreachable

=== Request ===
1. Please add the Omega centraltech-nonprod1 cluster's NAT/egress IP ranges
   to the MSK security groups and YCPI MSD transport policies.

2. Alternatively, please advise if there's a different recommended path
   for Omega EKS consumers (e.g., VPC PrivateLink).

=== Questions ===
1. Should Omega consumers use the YCPI path or a different endpoint?
2. What IP ranges need to be allow-listed for our cluster?
3. Are there any Athenz roles we need to be added to?

Thank you!
```

---

## Commands Reference

### Checking Pod Status

```bash
# Get pods
kubectl -n sre-kafka--consumer--nonprod--k8s get pods

# Get detailed pod info
kubectl -n sre-kafka--consumer--nonprod--k8s describe pod <pod-name>

# Get logs
kubectl -n sre-kafka--consumer--nonprod--k8s logs <pod-name> -c kafka-consumer

# Follow logs
kubectl -n sre-kafka--consumer--nonprod--k8s logs -f <pod-name> -c kafka-consumer
```

### Debugging Inside the Pod

```bash
# Exec into pod
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod-name> -c kafka-consumer -- /bin/bash

# Check Athenz certs
ls -la /var/run/athenz/
openssl x509 -in /var/run/athenz/service.cert.pem -subject -noout

# Test CKMS manually
python3 -c "
from ouroath.ckms.ysecure_plugin import YSecureOathCKMS
ckms = YSecureOathCKMS()
ckms.env = 'aws'
ckms.cert_filename = '/var/run/athenz/service.cert.pem'
ckms.key_filename = '/var/run/athenz/service.key.pem'
result = ckms.get_key('bigpanda.prod.api.sre', 'bigpanda.keys.sre')
print(f'Result: {result[:20]}...' if result else 'FAILED')
"

# Test DNS resolution
getent hosts broker1-public-bpimskprod-c21.yahooinc.com
getent hosts e1.ycpi.vip.dca.yahoo.com

# Test TCP connectivity (will fail until YCPI fix)
timeout 5 bash -c '</dev/tcp/e1.ycpi.vip.dca.yahoo.com/443' && echo SUCCESS || echo FAILED
```

### Network Policy Verification

```bash
# List network policies
kubectl -n sre-kafka--consumer--nonprod--k8s get networkpolicy

# Describe specific policy
kubectl -n sre-kafka--consumer--nonprod--k8s describe networkpolicy \
  sre.kafka-consumer-nonprod-k8s.kafka-consumer-development
```

### Cleanup Commands

```bash
# Delete stuck deployments
kubectl -n sre-kafka--consumer--nonprod--k8s delete deployment <deployment-name>

# Delete pods (will be recreated by deployment)
kubectl -n sre-kafka--consumer--nonprod--k8s delete pod <pod-name>

# Restart deployment
kubectl -n sre-kafka--consumer--nonprod--k8s rollout restart deployment <deployment-name>
```

### Copying Files from Pod

```bash
# Copy cert/key for local testing
kubectl -n sre-kafka--consumer--nonprod--k8s cp \
  <pod-name>:/var/run/athenz/service.cert.pem ./pod-cert.pem -c kafka-consumer

kubectl -n sre-kafka--consumer--nonprod--k8s cp \
  <pod-name>:/var/run/athenz/service.key.pem ./pod-key.pem -c kafka-consumer

# Test locally with ckms-remotecli
ckms-remotecli -env aws \
  -tlscert ./pod-cert.pem \
  -tlskey ./pod-key.pem \
  -group bigpanda.keys.sre \
  -key bigpanda.prod.api.sre
```

---

## Lessons Learned

### 1. Omega/Athenz Configuration

| Lesson | Details |
|--------|---------|
| **Enable ZTS identity providers** | For any Omega app using CKMS, enable "Kubernetes (Omega) launches instances for the service" AND "Allow ZTS as an identity provider for the service" in Athenz UI |
| **Use `appImage` not `base`** | When specifying your own Docker image in omega.yaml, use `appImage` field |
| **Config injection via appSrcDir** | Use `custom.appSrcDir` and `appTargetDir` to bake configs into the image; more reliable than ConfigMaps for static configs |
| **Custom probes need scripts** | If `probe: custom`, you MUST create `app_liveness` and `app_readiness` scripts |

### 2. CKMS Integration

| Lesson | Details |
|--------|---------|
| **Set environment explicitly** | Always set `ckms_client.env = 'aws'` in Python code; don't rely on environment variables alone |
| **Debug with logging** | Enable `logging.basicConfig(level=logging.DEBUG)` when troubleshooting CKMS |
| **Test from inside pod** | Copy certs out and test with `ckms-remotecli` locally to isolate pod vs library issues |

### 3. Network Connectivity

| Lesson | Details |
|--------|---------|
| **Not all network issues are K8s-level** | Kubernetes NetworkPolicy and Istio might be configured correctly, but external allow-lists (YCPI/MSD/Security Groups) can still block traffic |
| **YCPI fronts many services** | MSK and other services are often accessed via YCPI VIPs; understand the network path |
| **Terraform modules have defaults** | `tf-omega-app-network-policy` allows egress to 443/4443 by default |

### 4. Debugging Approach

| Lesson | Details |
|--------|---------|
| **Isolate layers systematically** | DNS → TCP → TLS → Application. Failure at each layer means different things |
| **Check logs at each container** | App container, istio-proxy, sia containers all have useful logs |
| **Document everything** | Keep detailed notes of errors, commands, and fixes for future reference |

---

## Appendix: Complete Configuration Reference

### A. omega.yaml (Current Working Version)

```yaml
# deploy_target/omega/omega.yaml
template: generic-app:stable
appName: kafka-consumer
maintainer: sre-team@yahooinc.com

# Athenz identity configuration
athensDomain: sre.kafka-consumer-nonprod-k8s
requireTLSCerts: true
identityv3: true

# Docker image
appImage: docker.ouroath.com:4443/yahoo-sre/yahoo.kafka_consumer:latest

# Custom health checks (our app doesn't serve HTTP)
probe: custom

# Config injection - bakes config into image
custom:
  appSrcDir: deploy_target/omega/config
  appTargetDir: /app/config

jobs:
  component:
    description: Development deployment
    clusterEnvironment: centraltech-nonprod1
    colo: use1
    athenzService: kafka-consumer-development
    autoscale:
      replicas: 1
    volumes:
    - name: kafka-consumer-history
      persistentVolumeClaim:
        claimName: kafka-consumer-history-dev
    volumeMounts:
    - mountPath: /data/automation_consumer
      name: kafka-consumer-history
```

### B. app_start Script

```bash
#!/bin/bash
set -ex

echo "=== Starting kafka_consumer ==="

# Wait for Athenz certificates
ATHENZ_CERT="/var/run/athenz/service.cert.pem"
ATHENZ_KEY="/var/run/athenz/service.key.pem"

MAX_WAIT=300
WAITED=0
while [ ! -f "$ATHENZ_CERT" ] || [ ! -f "$ATHENZ_KEY" ]; do
    echo "Waiting for Athenz certificates... ($WAITED seconds)"
    sleep 5
    WAITED=$((WAITED + 5))
    if [ $WAITED -ge $MAX_WAIT ]; then
        echo "ERROR: Athenz certificates not available after ${MAX_WAIT}s"
        exit 1
    fi
done
echo "Athenz certificates available!"

# Configuration
DEFAULT_CONFIG_FILE="/app/config/config.yaml"
CONFIG_FILE="${CONFIG_FILE:-$DEFAULT_CONFIG_FILE}"
export CONFIG_FILE
export LOG_LEVEL="${LOG_LEVEL:-INFO}"
export ACTION_HISTORY_FILE="/data/automation_consumer/automation_action_history.json"
export CKMS_ENV="aws"

# Verify config exists
if [ ! -f "$CONFIG_FILE" ]; then
    echo "ERROR: Config file not found at $CONFIG_FILE"
    exit 1
fi

# Create PID directory
PIDDIR="/var/run/kafka_consumer"
PIDFILE="${PIDDIR}/consumer.pid"
mkdir -p "$PIDDIR"

# Start the consumer
echo "Starting kafka_consumer..."
/opt/y/1.0/bin/kafka_consumer \
    --config "$CONFIG_FILE" \
    --log-level "$LOG_LEVEL" &

# Store PID for health checks
echo $! > "$PIDFILE"
echo "Started with PID $(cat "$PIDFILE")"

# Keep container running
wait
```

### C. app_liveness / app_readiness Scripts

```bash
#!/bin/bash
# Check if kafka_consumer process is running
PIDFILE="/var/run/kafka_consumer/consumer.pid"

if [ -f "$PIDFILE" ]; then
    PID=$(cat "$PIDFILE")
    if ps -p "$PID" > /dev/null; then
        exit 0
    else
        rm -f "$PIDFILE"
        exit 1
    fi
else
    exit 1
fi
```

### D. secret_manager.py (Key Section)

```python
def get_ckms_key(self, x509_cert: str, x509_key: str, ckms_key: str, ckms_key_group: str):
    """Get a CKMS key"""
    ckms_client = YSecureOathCKMS()
    ckms_client.env = 'aws'  # CRITICAL: Must be set explicitly
    ckms_client.cert_filename = x509_cert
    ckms_client.key_filename = x509_key

    try:
        ckms_res = ckms_client.get_key(ckms_key, ckms_key_group)
    except Exception as e:
        logging.error(f"CKMS exception for group={ckms_key_group}, key={ckms_key}: {e}")
        raise

    if ckms_res is None:
        err_str = f"Failed to get CKMS key: group={ckms_key_group}, key={ckms_key}"
        logging.error(err_str)
        raise Exception(err_str)

    return ckms_res
```

### E. Terraform Network Policy

```hcl
# terraform/athenz/sre.kafka-consumer-nonprod-k8s/network-policies.tf
locals {
  cloud_provider = "omega-aws"
  cloud_clusters = ["centraltech-nonprod1"]
}

module "tf-omega-app-network-policy" {
  source  = "edge.artifactory.ouroath.com:4443/omega-tf-modules__yahoo-cloud/tf-omega-app-network-policy/athenz"
  version = "~>0.0.7"

  athenz_domain   = local.domain
  omega_app_name  = "kafka-consumer"
  omega_app_ports = [4080]

  common_tcp_ingress_services = []
  tcp_ingress_services_per_environment = {
    "development" : [],
  }

  cloud_provider = local.cloud_provider
  cloud_clusters = local.cloud_clusters
}
```

---

## Document History

| Date | Author | Changes |
|------|--------|---------|
| 2026-01-29 | SRE Team | Initial comprehensive guide |

---

## Quick Reference Card

### 🔴 BLOCKED: What You're Waiting For

**MSK/YCPI Team** must add Omega cluster NAT IPs to network allow-lists.

### 🟢 READY: What's Working

- ✅ Application builds and deploys
- ✅ Configuration injection works
- ✅ CKMS secrets retrieved (all 4 keys)
- ✅ Health checks passing
- ✅ Kubernetes NetworkPolicy configured

### 📞 Who to Contact

| Issue | Team | Channel |
|-------|------|---------|
| YCPI/MSD allow-lists | YCPI Team | Jira + Slack #ycpi-help |
| MSK Security Groups | MSK Team (Thomas Neilly) | Jira + Slack #sre-tooling |
| Omega Platform | Omega Team | Slack #k8s-community |

### 🧪 Quick Verification Commands

```bash
# Check if YCPI is reachable (run after fix is applied)
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod> -c kafka-consumer -- \
  timeout 5 bash -c '</dev/tcp/e1.ycpi.vip.dca.yahoo.com/443' && echo SUCCESS || echo FAILED

# Check logs for "connected" or "subscribed"
kubectl -n sre-kafka--consumer--nonprod--k8s logs <pod> -c kafka-consumer | grep -i "connected\|subscribed"
```
