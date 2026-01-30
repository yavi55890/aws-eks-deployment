# Kafka MSK Connectivity Plan for Omega EKS Deployment

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current State](#current-state)
3. [Architecture Overview](#architecture-overview)
4. [Problem Analysis](#problem-analysis)
5. [What's Already Configured Correctly](#whats-already-configured-correctly)
6. [What Needs to Be Done](#what-needs-to-be-done)
7. [Configuration Options](#configuration-options)
8. [Implementation Plan](#implementation-plan)
9. [Verification Steps](#verification-steps)
10. [Appendix](#appendix)

---

## Executive Summary

The `kafka-consumer` application is deployed to **Omega EKS** (`omega-aws.centraltech-nonprod1.use1`) and needs to connect to **Amazon MSK** (Managed Streaming for Apache Kafka) brokers. While the application successfully retrieves credentials from CKMS and starts correctly, it **cannot establish network connectivity** to the Kafka brokers.

**Root Cause:** The MSK cluster is fronted by **YCPI (Yahoo Cloud Platform Infrastructure)** VIPs, and the Omega EKS cluster's egress IP ranges are **not included** in the MSK/YCPI network allow-lists (MSD policies, Security Groups, WAKL).

**Key Finding:** This is **NOT** an omega.yaml or Kubernetes NetworkPolicy issue. The fix must come from the **MSK/YCPI infrastructure owners**.

---

## Current State

### Application Details

| Property | Value |
|----------|-------|
| Application | `kafka-consumer` |
| Athenz Domain (nonprod) | `sre.kafka-consumer-nonprod-k8s` |
| Athenz Service | `kafka-consumer-development` |
| Omega Cluster | `omega-aws.centraltech-nonprod1.use1` |
| Namespace | `sre-kafka--consumer--nonprod--k8s` |

### MSK Broker Details

| Property | Value |
|----------|-------|
| MSK Cluster | `bpi-msk-prod` (BigPanda MSK) |
| Authentication | SASL/SCRAM (via CKMS) |
| Port | `9198` (SASL/SCRAM over TLS) |

**Broker Endpoints:**
```
b-1-public.alerts.kafka.us-east-1.amazonaws.com:9198
b-2-public.alerts.kafka.us-east-1.amazonaws.com:9198
b-3-public.alerts.kafka.us-east-1.amazonaws.com:9198
```

### Error Observed

```
%3|...|FAIL|... sasl_ssl://broker2-public-bpimskprod-c21.yahooinc.com:443/bootstrap:
  Failed to connect to broker at [e1.ycpi.vip.dca.yahoo.com]:443: Network is unreachable
%3|...|FAIL|... sasl_ssl://broker1-public-bpimskprod-c21.yahooinc.com:443/bootstrap:
  Failed to connect to broker at [e1.ycpi.vip.dca.yahoo.com]:443: Network is unreachable
```

**Translation:** The Kafka client (librdkafka) resolves the broker hostnames to YCPI VIP addresses, but packets cannot reach those VIPs from the Omega cluster.

---

## Architecture Overview

### Network Path: Omega Pod → MSK Broker

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OMEGA EKS CLUSTER                                  │
│                    (centraltech-nonprod1.use1)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  kafka-consumer Pod                                                  │    │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐              │    │
│  │  │ App Container│    │Istio Sidecar│    │ (optional)  │              │    │
│  │  │             │───▶│   (Envoy)   │───▶│             │              │    │
│  │  └─────────────┘    └─────────────┘    └─────────────┘              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Kubernetes NetworkPolicy (MSD)                          │    │
│  │              ✅ Allows egress to ports 443, 4443, 9198               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              AWS VPC NAT Gateway                                     │    │
│  │              (Egress IP: x.x.x.x)                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ Internet / AWS Network
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              YCPI LAYER                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  YCPI VIP: e1.ycpi.vip.dca.yahoo.com                                │    │
│  │                                                                      │    │
│  │  ❌ BLOCKED: Omega NAT IPs not in allow-list                        │    │
│  │                                                                      │    │
│  │  MSD Transport Policy checks:                                        │    │
│  │  - Source IP in allowed prefix list?                                 │    │
│  │  - Athenz principal authorized?                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │ (if allowed)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AMAZON MSK CLUSTER                                 │
│                           (bpi-msk-prod)                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  AWS Security Group                                                  │    │
│  │  - Allows YCPI IP ranges on port 9196/9198                          │    │
│  │  - Does NOT include Omega cluster egress IPs                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Brokers:                                                                    │
│  - b-1-public.alerts.kafka.us-east-1.amazonaws.com:9198                     │
│  - b-2-public.alerts.kafka.us-east-1.amazonaws.com:9198                     │
│  - b-3-public.alerts.kafka.us-east-1.amazonaws.com:9198                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Components Explained

#### 1. YCPI (Yahoo Cloud Platform Infrastructure)

YCPI provides network routing and load balancing for Yahoo services. MSK clusters are **fronted by YCPI VIPs** to:
- Provide a stable DNS endpoint
- Handle routing between on-prem and cloud
- Enforce access control via MSD policies

When you resolve `broker1-public-bpimskprod-c21.yahooinc.com`, it returns a YCPI VIP like `e1.ycpi.vip.dca.yahoo.com`.

#### 2. MSD (Micro-Segmentation for Distributed systems)

MSD is Yahoo's network segmentation framework that uses **Athenz identities** to control network access. It creates:
- **Transport Policies**: Define which services can communicate
- **Network Policies**: Kubernetes-level firewall rules
- **WAKL (Web Application Key List)**: IP-based allow-lists

#### 3. Istio Service Mesh

On Omega AWS EKS, Istio is enabled by default with `STRICT_MTLS`. The Istio sidecar:
- Intercepts all traffic
- Enforces mTLS between services
- Can restrict egress via ServiceEntry/Sidecar resources

**However:** In this case, Istio is **NOT the blocker**. The traffic is failing at the network layer before reaching YCPI.

#### 4. Kubernetes NetworkPolicy

Created via the `tf-omega-app-network-policy` Terraform module. This controls:
- **Ingress**: Which pods can send traffic TO your app
- **Egress**: Which external hosts your app can reach

---

## Problem Analysis

### What's Working

| Component | Status | Evidence |
|-----------|--------|----------|
| CKMS Secret Retrieval | ✅ Working | App retrieves SCRAM credentials |
| Pod Startup | ✅ Working | Container starts successfully |
| DNS Resolution | ✅ Working | Broker hostnames resolve |
| K8s NetworkPolicy | ✅ Configured | Allows egress to 443/4443 |
| Athenz Identity | ✅ Provisioned | Certs mounted at `/var/run/athenz/` |

### What's NOT Working

| Component | Status | Issue |
|-----------|--------|-------|
| TCP Connection to YCPI | ❌ Failed | `Network is unreachable` |
| YCPI/MSD Allow-list | ❌ Missing | Omega IPs not in prefix list |
| MSK Security Group | ❌ Missing | Omega IPs not allowed |

### Root Cause

The Omega EKS cluster (`centraltech-nonprod1`) uses NAT Gateways for outbound internet traffic. These NAT Gateway IP addresses are **not included** in:

1. **YCPI MSD Transport Policy** prefix lists
2. **MSK AWS Security Group** ingress rules
3. **WAKL** (Web Application Key List) entries

Therefore, when the kafka-consumer pod tries to connect to the YCPI VIP, the connection is **dropped at the network layer**.

---

## What's Already Configured Correctly

### 1. Omega.yaml Configuration

```yaml
# deploy_target/omega/omega.yaml
template: generic-app:stable
appName: kafka-consumer
athensDomain: sre.kafka-consumer-nonprod-k8s

# Athenz identity enabled
requireTLSCerts: true
identityv3: true
```

**Status:** ✅ Correct - Athenz certificates are provisioned for CKMS access.

### 2. Terraform Network Policy

```hcl
# terraform/athenz/sre.kafka-consumer-nonprod-k8s/network-policies.tf
module "tf-omega-app-network-policy" {
  source  = "edge.artifactory.ouroath.com:4443/omega-tf-modules__yahoo-cloud/tf-omega-app-network-policy/athenz"
  version = "~>0.0.7"

  athenz_domain   = local.domain
  omega_app_name  = "kafka-consumer"
  omega_app_ports = [4080]

  # Default egress allows: 443, 4443, 80, 4080, 53
  cloud_provider = local.cloud_provider
  cloud_clusters = local.cloud_clusters
}
```

**Status:** ✅ Correct - Default egress rules allow port 443/4443.

### 3. MSD Allowlist Module

```hcl
# terraform/athenz/common/common-network-policies.tf
module "tf-omega-msd-allowlist" {
  source  = "edge.artifactory.ouroath.com:4443/omega-tf-modules__yahoo-cloud/tf-omega-msd-allowlist/athenz"
  version = "~>0.0.6"

  athenz_domain = local.domain
}
```

**Status:** ✅ Correct - MSD allowlist is configured for the domain.

---

## What Needs to Be Done

### Summary of Required Actions

| # | Action | Owner | Priority |
|---|--------|-------|----------|
| 1 | Add Omega egress IPs to MSK/YCPI allow-lists | MSK/YCPI Team | **Critical** |
| 2 | (Optional) Add port 9198 to custom egress rules | SRE Team | Low |
| 3 | (Optional) Configure Istio egress if using service mesh | SRE Team | Low |
| 4 | (Optional) Create ns-config.yaml for Secure Egress | SRE Team | Low |

### Critical Action: MSK/YCPI Network Allow-list

This is the **only** action that will fix the connectivity issue.

**Contact:** MSK/YCPI Team (SRE Tooling - Thomas Neilly / YCPI team)

**Request Details:**

```
Subject: Request to add Omega EKS cluster to MSK/YCPI allow-list

MSK Cluster: bpi-msk-prod
Broker Endpoints:
  - b-1-public.alerts.kafka.us-east-1.amazonaws.com:9198
  - b-2-public.alerts.kafka.us-east-1.amazonaws.com:9198
  - b-3-public.alerts.kafka.us-east-1.amazonaws.com:9198

YCPI VIP observed: e1.ycpi.vip.dca.yahoo.com:443

Source Details:
  - Omega Cluster: omega-aws.centraltech-nonprod1.use1
  - Namespace: sre-kafka--consumer--nonprod--k8s
  - Athenz Domain: sre.kafka-consumer-nonprod-k8s
  - Athenz Service: kafka-consumer-development
  - Full Principal: sre.kafka-consumer-nonprod-k8s.kafka-consumer-development

Error: "Network is unreachable" when connecting to YCPI VIP from consumer pods

Request:
1. Please add the Omega centraltech-nonprod1 cluster's NAT/egress IP ranges
   to the MSK security groups and YCPI MSD transport policies.
2. Alternatively, please advise if there's a different recommended path
   for Omega EKS consumers (e.g., VPC PrivateLink, direct VPC peering).

Questions:
1. Should Omega consumers use the YCPI path or a different endpoint?
2. What IP ranges need to be allow-listed for our cluster?
3. Are there any Athenz roles we need to be added to for access?
```

---

## Configuration Options

### Option A: YCPI Path (Current Approach) - RECOMMENDED

**Description:** Continue using the YCPI-fronted MSK endpoints with SASL/SCRAM authentication.

**Pros:**
- Already partially configured
- Standard Yahoo approach for on-prem consumers
- Uses existing SCRAM credentials from CKMS

**Cons:**
- Requires MSK/YCPI team to update allow-lists
- Dependent on YCPI infrastructure

**Required Changes:**
1. MSK/YCPI team adds Omega NAT IPs to allow-lists
2. No changes needed on our side

### Option B: VPC PrivateLink (Alternative)

**Description:** Use AWS PrivateLink to create private endpoints for MSK within the Omega VPC.

**Pros:**
- Direct VPC-to-VPC connectivity
- No YCPI dependency
- Lower latency

**Cons:**
- Requires AWS infrastructure changes
- May need different MSK endpoints
- Potentially different authentication method (IAM)

**Required Changes:**
1. MSK team creates PrivateLink endpoints
2. Update broker URLs in consumer config
3. Possibly switch from SCRAM to IAM auth

### Option C: Direct Public Endpoint (Not Recommended)

**Description:** Bypass YCPI and connect directly to MSK public endpoints.

**Pros:**
- Simpler network path

**Cons:**
- May not be allowed by security policy
- Exposes MSK to broader internet
- Still requires Security Group changes

---

## Implementation Plan

### Phase 1: Immediate (MSK/YCPI Team Action Required)

#### Step 1.1: Gather Omega Cluster Egress IPs

Run from your workstation with kubectl access:

```bash
# Get the cluster's NAT Gateway IPs (ask Omega team if not available)
kubectl get nodes -o wide | grep -i external
```

Or contact the Omega platform team for the NAT Gateway IP ranges for `centraltech-nonprod1`.

#### Step 1.2: Submit Request to MSK/YCPI Team

Create a Jira ticket or Slack the MSK/YCPI team with the details from [What Needs to Be Done](#critical-action-mskypi-network-allow-list).

#### Step 1.3: Verification

Once the allow-list is updated, verify connectivity:

```bash
# From the kafka-consumer pod
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod-name> -c app -- \
  bash -c "timeout 5 bash -c '</dev/tcp/e1.ycpi.vip.dca.yahoo.com/443' && echo 'SUCCESS' || echo 'FAILED'"
```

### Phase 2: Optional Hardening (After Connectivity Works)

#### Step 2.1: Add Custom Egress Port (9198)

If the default ports (443/4443) don't cover your Kafka port, update the network policy:

**File:** `terraform/athenz/sre.kafka-consumer-nonprod-k8s/network-policies.tf`

```hcl
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

  # Add custom egress for Kafka port 9198
  custom_egress_rules_per_environment = {
    "development" = [
      {
        port     = 9198
        protocol = "TCP"
      }
    ]
  }

  cloud_provider = local.cloud_provider
  cloud_clusters = local.cloud_clusters
}
```

#### Step 2.2: Configure Istio Egress (If Required)

If Istio is blocking external traffic, add egress hosts to `omega.yaml`:

```yaml
sidecars:
  istio:
    enabled: true
    egress:
    - hosts:
      - "*.alerts.kafka.us-east-1.amazonaws.com"
      - "*.ycpi.vip.dca.yahoo.com"
```

#### Step 2.3: Create Secure Egress Configuration (Optional)

For stricter egress control, create a namespace config pipeline:

**File:** `deploy_target/omega/ns-config.yaml`

```yaml
template: ns-config:stable
athensDomain: sre.kafka-consumer-nonprod-k8s
maintainer: sre-team@yahooinc.com

egress:
  external: {}

jobs:
  pull-request:
    clusterEnvironment: centraltech-nonprod1
    colo: use1
  component:
    clusterEnvironment: centraltech-nonprod1
    colo: use1
    egress:
      external:
        "kafka-msk-brokers":
          hosts:
          - "b-1-public.alerts.kafka.us-east-1.amazonaws.com"
          - "b-2-public.alerts.kafka.us-east-1.amazonaws.com"
          - "b-3-public.alerts.kafka.us-east-1.amazonaws.com"
          ports:
          - number: 9198
            protocol: TCP
        "ycpi-vip":
          hosts:
          - "e1.ycpi.vip.dca.yahoo.com"
          ports:
          - number: 443
            protocol: TCP
```

---

## Verification Steps

### Pre-Fix Diagnostics

Run these commands to document the current state before fixes are applied:

```bash
# 1. Check DNS resolution
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod> -c app -- \
  getent hosts b-1-public.alerts.kafka.us-east-1.amazonaws.com

kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod> -c app -- \
  getent hosts e1.ycpi.vip.dca.yahoo.com

# 2. Test TCP connectivity (will fail before fix)
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod> -c app -- \
  bash -c "timeout 5 bash -c '</dev/tcp/e1.ycpi.vip.dca.yahoo.com/443' || echo 'FAILED: Network unreachable'"

# 3. Check Istio sidecar logs for any blocks
kubectl -n sre-kafka--consumer--nonprod--k8s logs <pod> -c istio-proxy | grep -i "denied\|blocked"

# 4. Verify NetworkPolicy allows egress
kubectl -n sre-kafka--consumer--nonprod--k8s get networkpolicy -o yaml
```

### Post-Fix Verification

After MSK/YCPI team updates the allow-lists:

```bash
# 1. Test TCP connectivity (should succeed)
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod> -c app -- \
  bash -c "timeout 5 bash -c '</dev/tcp/e1.ycpi.vip.dca.yahoo.com/443' && echo 'SUCCESS'"

# 2. Test Kafka broker directly
kubectl -n sre-kafka--consumer--nonprod--k8s exec -it <pod> -c app -- \
  bash -c "timeout 5 bash -c '</dev/tcp/b-1-public.alerts.kafka.us-east-1.amazonaws.com/9198' && echo 'SUCCESS'"

# 3. Check consumer logs for successful connection
kubectl -n sre-kafka--consumer--nonprod--k8s logs <pod> -c app | grep -i "connected\|subscribed"

# 4. Verify message consumption
kubectl -n sre-kafka--consumer--nonprod--k8s logs <pod> -c app | tail -50
```

---

## Appendix

### A. Athenz Principal Format

Your application's Athenz principal is constructed as:

```
<athenz-domain>.<athenz-service>
```

| Environment | Athenz Domain | Service | Full Principal |
|-------------|---------------|---------|----------------|
| Development | `sre.kafka-consumer-nonprod-k8s` | `kafka-consumer-development` | `sre.kafka-consumer-nonprod-k8s.kafka-consumer-development` |
| Production | `sre.kafka-consumer-k8s` | `kafka-consumer-production` | `sre.kafka-consumer-k8s.kafka-consumer-production` |

### B. Relevant Documentation Links

| Topic | URL |
|-------|-----|
| Service Mesh | https://ouryahoo.atlassian.net/wiki/spaces/KUBE/pages/294850741/Service+Mesh |
| Secure Egress | https://ouryahoo.atlassian.net/wiki/spaces/KUBE/pages/294950432/Secure+Egress |
| Egress Gateway | https://ouryahoo.atlassian.net/wiki/spaces/KUBE/pages/827621572/Egress+Gateway |
| Routing.yaml | https://ouryahoo.atlassian.net/wiki/spaces/KUBE/pages/294881690/Routing.yaml |
| Network Policies | https://ouryahoo.atlassian.net/wiki/spaces/KUBE/pages/295437421 |

### C. Terraform Module Reference

| Module | Purpose | Documentation |
|--------|---------|---------------|
| `tf-omega-app-network-policy` | Creates K8s NetworkPolicy for app egress/ingress | Artifactory |
| `tf-omega-msd-allowlist` | Configures MSD allowlist for domain | Artifactory |
| `tf-omega-athenz-msd` | Creates MSD transport policies | Artifactory |

### D. Key Contacts

| Team | Responsibility | Contact |
|------|----------------|---------|
| MSK/YCPI | Network allow-lists, MSD policies | SRE Tooling (Thomas Neilly) |
| Omega Platform | Cluster networking, NAT IPs | #k8s-community Slack |
| BigPanda MSK | MSK cluster ownership | BigPanda SRE |

### E. Glossary

| Term | Definition |
|------|------------|
| **YCPI** | Yahoo Cloud Platform Infrastructure - network routing layer |
| **MSD** | Micro-Segmentation for Distributed systems - network access control |
| **WAKL** | Web Application Key List - IP-based allow-lists |
| **VIP** | Virtual IP - load-balanced endpoint |
| **NAT Gateway** | AWS component that provides outbound internet access for private subnets |
| **Istio** | Service mesh that handles mTLS and traffic management |
| **ServiceEntry** | Istio resource that allows mesh to access external services |
| **NetworkPolicy** | Kubernetes resource for pod-level firewall rules |

---

## Document History

| Date | Author | Changes |
|------|--------|---------|
| 2026-01-29 | SRE Team | Initial document creation |

---

## Summary

**The Problem:** Kafka consumer cannot reach MSK brokers because Omega EKS cluster's egress IPs are not in the YCPI/MSK network allow-lists.

**The Solution:** MSK/YCPI team must add the Omega cluster's NAT Gateway IP ranges to their MSD transport policies and AWS Security Groups.

**Your Action Items:**
1. Contact MSK/YCPI team with the request details provided above
2. Wait for network allow-list update
3. Verify connectivity using the provided test commands
4. (Optional) Add port 9198 to custom egress rules if needed

**What You DON'T Need to Do:**
- No changes to omega.yaml for network connectivity
- No Istio configuration changes (Istio is not the blocker)
- No Kubernetes NetworkPolicy changes (already allows required ports)
