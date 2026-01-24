# Omega Operations Guide: Running Kafka Consumers

This document explains how the kafka_consumer runs on Omega (Kubernetes), how it stays running, and how to manage multiple consumers with different configurations.

---

## Table of Contents

1. [How the Consumer Runs on Kubernetes](#how-the-consumer-runs-on-kubernetes)
2. [What Keeps It Running](#what-keeps-it-running)
3. [Running Multiple Consumers with Different Configs](#running-multiple-consumers-with-different-configs)
4. [Production Architecture](#production-architecture)
5. [Important Considerations](#important-considerations)
6. [Manual Operations](#manual-operations)

---

## How the Consumer Runs on Kubernetes

### The Pod Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        POD LIFECYCLE ON OMEGA                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. SCHEDULING                                                              │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ Kubernetes scheduler places pod on a node in corp.bf1           │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  2. INIT PHASE (up to 20 minutes)                                          │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • Pull Docker image from registry                               │    │
│     │ • Mount PVC (/data/automation_consumer)                         │    │
│     │ • Athenz sidecar provisions certificates to /var/run/athenz/    │    │
│     │ • app_configure runs (creates directories)                      │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  3. STARTUP (app_start script)                                             │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • Wait for Athenz certs (up to 5 min)                           │    │
│     │ • Verify config file exists                                     │    │
│     │ • Start kafka_consumer process in background                    │    │
│     │ • Write PID to /var/run/kafka_consumer/consumer.pid             │    │
│     │ • Script waits (keeps container alive)                          │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  4. RUNNING (continuous)                                                    │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │                                                                 │    │
│     │   kafka_consumer process:                                       │    │
│     │   • Connects to Kafka brokers                                   │    │
│     │   • Subscribes to topics                                        │    │
│     │   • Polls for messages in infinite loop                         │    │
│     │   • Processes incidents, runs actions                           │    │
│     │   • Writes action history to PVC                                │    │
│     │                                                                 │    │
│     │   Kubernetes monitors:                                          │    │
│     │   • Liveness probe (every 30s) - is process alive?              │    │
│     │   • Readiness probe (every 30s) - is process ready?             │    │
│     │                                                                 │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  5. FAILURE RECOVERY (automatic)                                           │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ If liveness probe fails 3 times:                                │    │
│     │   → Kubernetes kills the pod                                    │    │
│     │   → Kubernetes starts a NEW pod                                 │    │
│     │   → PVC data persists (action history retained)                 │    │
│     │   → Consumer resumes from last Kafka offset                     │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  6. GRACEFUL SHUTDOWN (on deploy/scale-down)                               │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │ • app_prestop script runs                                       │    │
│     │ • Sends SIGTERM to consumer                                     │    │
│     │ • Consumer commits Kafka offsets, closes connections            │    │
│     │ • Waits up to 120s (terminationGracePeriodSeconds)              │    │
│     │ • Pod terminates                                                │    │
│     └─────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Startup Script Flow

The `app_start` script (`deploy_target/omega/scripts/app_start`) does the following:

```bash
# 1. Wait for Athenz certificates
while [ ! -f "/var/run/athenz/service.cert.pem" ]; do
    sleep 5
done

# 2. Set environment variables
export CONFIG_FILE="/app/config/config.yaml"
export ACTION_HISTORY_FILE="/data/automation_consumer/automation_action_history.json"

# 3. Start the consumer
/opt/y/1.0/bin/kafka_consumer --config "$CONFIG_FILE" &

# 4. Store PID for health checks
echo $! > /var/run/kafka_consumer/consumer.pid

# 5. Wait (keeps container running)
wait
```

---

## What Keeps It Running

Kubernetes guarantees your pod stays running through several mechanisms:

### Replica Guarantee

```yaml
# In omega.yaml
autoscale:
  minReplicas: 1
  maxReplicas: 1  # Single instance
```

- **Pod crashes** → K8s automatically restarts it
- **Node dies** → K8s reschedules pod to another node
- **New deployment** → K8s does rolling update (stops old, starts new)

### Health Probes

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| **Liveness** | Is the process alive? | Kill pod, restart |
| **Readiness** | Is the process ready? | Remove from service (no traffic) |

The probes are shell scripts that check if the consumer process is running:

```bash
# liveness.sh - checks if process exists
PID=$(cat /var/run/kafka_consumer/consumer.pid)
kill -0 "$PID"  # Returns 0 if process exists

# readiness.sh - checks process + uptime
UPTIME=$(ps -p "$PID" -o etimes=)
[ "$UPTIME" -ge 30 ]  # Ready after 30 seconds
```

### Persistent Storage (PVC)

Action history survives pod restarts:

```
Pod Crash/Restart
       │
       ▼
┌──────────────────┐
│   PVC Storage    │  ← Data persists across restarts
│                  │
│ automation_      │
│ action_history   │
│ .json            │
└──────────────────┘
       │
       ▼
New Pod Starts
       │
       ▼
Reads existing history
(won't re-run same actions)
```

---

## Running Multiple Consumers with Different Configs

### Approach Comparison

| Approach | Pros | Cons |
|----------|------|------|
| **Multiple Jobs in omega.yaml** | Single file, easy to manage | Can get cluttered |
| **Separate omega.yaml files** | Clear separation | More files to maintain |
| **Environment Variable Selection** | Flexible, one deployment | All configs baked into image |
| **ConfigMaps** | Dynamic config, no redeploy | More K8s complexity |

### Recommended: Multiple Jobs in omega.yaml

Update `deploy_target/omega/omega.yaml`:

```yaml
# Base configuration shared by all consumers
template: generic-app:stable
appName: kafka-consumer
athensDomain: sre
base: docker.ouroath.com:4443/yahoo/kafka_consumer

# ... shared settings (probes, resources, etc.) ...

jobs:
  # Consumer 1: Handles Oozie alerts
  oozie-consumer-dev:
    environment: development
    colo: corp.bf1
    env:
      CONFIG_FILE: /app/config/oozie-config.yaml
      CONSUMER_NAME: oozie-consumer
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-oozie-dev
        mountPath: /data/automation_consumer

  oozie-consumer-prod:
    environment: production
    colo: corp.bf1
    env:
      CONFIG_FILE: /app/config/oozie-config.yaml
      CONSUMER_NAME: oozie-consumer
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-oozie-prod
        mountPath: /data/automation_consumer

  # Consumer 2: Handles Grid alerts
  grid-consumer-dev:
    environment: development
    colo: corp.bf1
    env:
      CONFIG_FILE: /app/config/grid-config.yaml
      CONSUMER_NAME: grid-consumer
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-grid-dev
        mountPath: /data/automation_consumer

  grid-consumer-prod:
    environment: production
    colo: corp.bf1
    env:
      CONFIG_FILE: /app/config/grid-config.yaml
      CONSUMER_NAME: grid-consumer
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-grid-prod
        mountPath: /data/automation_consumer

  # Consumer 3: Handles Network alerts
  network-consumer-prod:
    environment: production
    colo: corp.bf1
    env:
      CONFIG_FILE: /app/config/network-config.yaml
      CONSUMER_NAME: network-consumer
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-network-prod
        mountPath: /data/automation_consumer
```

### Config Files Structure

Create corresponding config files:

```
config/
├── oozie-config.yaml      # Oozie-specific incident_conditions
├── grid-config.yaml       # Grid-specific incident_conditions
└── network-config.yaml    # Network-specific incident_conditions
```

Each config file has different `incident_conditions`:

```yaml
# oozie-config.yaml
kafka:
  consumer_group_id: sre-oozie-consumer-prod  # Unique per consumer!
  topics: [sre]

incident_conditions:
  - active: true
    tags:
      alert_type: oozie
    alert_actions:
      - handle_oozie
```

```yaml
# grid-config.yaml
kafka:
  consumer_group_id: sre-grid-consumer-prod   # Different group ID!
  topics: [sre]

incident_conditions:
  - active: true
    tags:
      alert_type: grid
    alert_actions:
      - handle_grid
```

### Screwdriver Jobs for Multiple Consumers

Update `screwdriver.yaml`:

```yaml
jobs:
    # ... existing build jobs ...

    # Deploy Oozie consumer to dev
    oozie-consumer-dev:
        requires: [docker]
        template: omega/k8s-generic@latest
        environment:
            OMEGA_JOB: oozie-consumer-dev
            PUBLISH: true

    # Deploy Grid consumer to dev
    grid-consumer-dev:
        requires: [docker]
        template: omega/k8s-generic@latest
        environment:
            OMEGA_JOB: grid-consumer-dev
            PUBLISH: true

    # Deploy all prod consumers (manual trigger)
    deploy-production:
        requires: [~sd@1132405:oozie-consumer-dev, ~sd@1132405:grid-consumer-dev]
        template: omega/k8s-generic@latest
        environment:
            PUBLISH: true
```

---

## Production Architecture

### Visual Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OMEGA CLUSTER (corp.bf1)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Namespace: sre                                                            │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   ┌─────────────────────────┐    ┌─────────────────────────┐       │  │
│   │   │   oozie-consumer pod    │    │    grid-consumer pod    │       │  │
│   │   │                         │    │                         │       │  │
│   │   │  CONFIG: oozie-config   │    │  CONFIG: grid-config    │       │  │
│   │   │  TOPICS: [sre]          │    │  TOPICS: [sre]          │       │  │
│   │   │  GROUP: oozie-consumer  │    │  GROUP: grid-consumer   │       │  │
│   │   │                         │    │                         │       │  │
│   │   │  incident_conditions:   │    │  incident_conditions:   │       │  │
│   │   │  - Oozie alerts only    │    │  - Grid alerts only     │       │  │
│   │   │                         │    │                         │       │  │
│   │   └───────────┬─────────────┘    └───────────┬─────────────┘       │  │
│   │               │                              │                      │  │
│   │               ▼                              ▼                      │  │
│   │   ┌─────────────────────────┐    ┌─────────────────────────┐       │  │
│   │   │  PVC: oozie-history     │    │   PVC: grid-history     │       │  │
│   │   │  (action history)       │    │   (action history)      │       │  │
│   │   └─────────────────────────┘    └─────────────────────────┘       │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│                              │                                              │
│                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        KAFKA (MSK)                                   │  │
│   │                                                                      │  │
│   │   Topic: sre ──┬───────▶ oozie-consumer (filters for oozie alerts)  │  │
│   │                │                                                     │  │
│   │                └───────▶ grid-consumer (filters for grid alerts)    │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────────────────┐
│   BigPanda   │────▶│    Kafka     │────▶│      Omega Consumers             │
│              │     │              │     │                                  │
│  Incidents   │     │  Topic: sre  │     │  oozie-consumer:                 │
│  & Alerts    │     │              │     │    filters: alert_type=oozie     │
│              │     │              │     │    actions: handle_oozie         │
│              │     │              │     │                                  │
│              │     │              │     │  grid-consumer:                  │
│              │     │              │     │    filters: alert_type=grid      │
│              │     │              │     │    actions: handle_grid          │
└──────────────┘     └──────────────┘     └──────────────────────────────────┘
```

---

## Important Considerations

### 1. Consumer Group IDs Must Be Unique

Each consumer needs a unique `consumer_group_id` in its config:

```yaml
# oozie-config.yaml
kafka:
  consumer_group_id: sre-oozie-consumer-prod  # Unique!
  topics: [sre]

# grid-config.yaml
kafka:
  consumer_group_id: sre-grid-consumer-prod   # Different!
  topics: [sre]
```

**Why?** If two consumers share the same group ID, Kafka treats them as one consumer group and load-balances messages between them (each only gets half the messages).

### 2. Action History Files Must Be Separate

Each consumer needs its own PVC to avoid conflicts:

```yaml
pvcMounts:
  - claimName: kafka-consumer-oozie-prod   # Separate PVC per consumer
    mountPath: /data/automation_consumer
```

**Why?** The `FileBackedDict` for action history would have write conflicts if multiple consumers tried to write to the same file.

### 3. Shared Topic vs Separate Topics

| Approach | When to Use |
|----------|-------------|
| **Shared Topic** | Consumers filter by `incident_conditions` |
| **Separate Topics** | BigPanda routes different alerts to different topics |

**Shared Topic (Recommended for Most Cases):**
- All consumers read from `sre` topic
- Each filters by different `incident_conditions`
- Simpler Kafka setup
- Each consumer reads ALL messages but only processes matches

**Separate Topics:**
- BigPanda routes Oozie alerts to `sre-oozie`, Grid alerts to `sre-grid`
- Each consumer subscribes only to its topic
- More efficient (less filtering), but requires BigPanda configuration

### 4. Resource Allocation

Consider adjusting resources based on message volume:

```yaml
# High-volume consumer
resources:
  requests:
    cpu: 2
    memory: 4Gi
  limits:
    memory: 8Gi

# Low-volume consumer
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    memory: 2Gi
```

---

## Manual Operations

### Viewing Pods

```bash
# List all consumer pods
kubectl get pods -n sre -l app=kafka-consumer

# Get detailed pod info
kubectl describe pod -n sre <pod-name>

# Watch pod status in real-time
kubectl get pods -n sre -l app=kafka-consumer -w
```

### Viewing Logs

```bash
# View logs for specific consumer
kubectl logs -f -n sre deployment/oozie-consumer-prod

# View logs with timestamps
kubectl logs -f -n sre deployment/oozie-consumer-prod --timestamps

# View last 100 lines
kubectl logs -n sre deployment/oozie-consumer-prod --tail=100

# View logs from previous crashed container
kubectl logs -n sre deployment/oozie-consumer-prod --previous
```

### Restarting Consumers

```bash
# Graceful restart (triggers shutdown + new pod)
kubectl rollout restart -n sre deployment/oozie-consumer-prod

# Check rollout status
kubectl rollout status -n sre deployment/oozie-consumer-prod

# Rollback to previous version
kubectl rollout undo -n sre deployment/oozie-consumer-prod
```

### Debugging

```bash
# Shell into a pod
kubectl exec -it -n sre deployment/oozie-consumer-prod -- bash

# Check action history
kubectl exec -n sre deployment/oozie-consumer-prod -- \
  cat /data/automation_consumer/automation_action_history.json

# Check Athenz certificates
kubectl exec -n sre deployment/oozie-consumer-prod -- \
  ls -la /var/run/athenz/

# Check environment variables
kubectl exec -n sre deployment/oozie-consumer-prod -- env | grep -E "CONFIG|LOG|ACTION"

# Check if process is running
kubectl exec -n sre deployment/oozie-consumer-prod -- \
  ps aux | grep kafka_consumer

# Check PVC mount
kubectl exec -n sre deployment/oozie-consumer-prod -- \
  df -h /data/automation_consumer
```

### Scaling (Not Recommended for This App)

Due to file-based state (`FileBackedDict`), scaling beyond 1 replica is **not recommended**:

```yaml
autoscale:
  minReplicas: 1
  maxReplicas: 1  # Keep at 1!
```

If you need higher throughput, run **multiple consumers with different configs** instead of scaling one consumer.

---

## Quick Reference

| Question | Answer |
|----------|--------|
| **How does it start?** | `app_start` script waits for Athenz certs, then runs `kafka_consumer --config ...` |
| **How does it stay running?** | Kubernetes guarantees `minReplicas: 1`, auto-restarts on failure |
| **How to run multiple consumers?** | Multiple jobs in `omega.yaml`, each with different `CONFIG_FILE` env var |
| **What about different configs?** | Create separate config files, reference via environment variable |
| **What if one crashes?** | K8s restarts it, PVC preserves action history, resumes from Kafka offset |
| **How to restart?** | `kubectl rollout restart -n sre deployment/<consumer-name>` |
| **How to view logs?** | `kubectl logs -f -n sre deployment/<consumer-name>` |
| **How to debug?** | `kubectl exec -it -n sre deployment/<consumer-name> -- bash` |
