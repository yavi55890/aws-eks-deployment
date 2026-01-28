# Omega EKS Deployment Guide

This guide explains how the kafka-consumer application is deployed to Omega Kubernetes (EKS) clusters using Screwdriver CI/CD.

## Overview

The deployment uses:
- **Screwdriver CI/CD** for build pipeline orchestration
- **Omega/k8s-generic template** for Kubernetes deployments
- **AWS EKS clusters** via Omega's KDNC infrastructure

### Target Clusters

| Environment | Cluster | Namespace |
|-------------|---------|-----------|
| Development | `omega-aws.centraltech-nonprod1.use1` | `sre.kafka-consumer-nonprod-k8s` |
| Production | `omega-aws.centraltech-prod1.use1` | `sre.kafka-consumer-prod-k8s` |

---

## Directory Structure

```
deploy_target/
└── omega/
    ├── omega.yaml              # Main Omega configuration
    ├── config/                 # Config files (created by SD job)
    │   └── config.yaml
    └── scripts/
        ├── app_configure       # Runs during image build
        ├── app_start           # Runs when container starts
        └── app_prestop         # Runs before container termination
```

---

## omega.yaml Configuration

The `deploy_target/omega/omega.yaml` file defines how the application is deployed to Kubernetes.

### Core Settings

```yaml
template: generic-app:stable    # Omega deployment template
appName: kafka-consumer         # Application name in Omega
maintainer: sre-team@yahooinc.com
athensDomain: sre               # Athenz domain for authentication
```

### Docker Image

```yaml
base: docker.ouroath.com:4443/yahoo/kafka_consumer
```

The Docker image is built by Screwdriver and pushed to Artifactory. The `base` field tells Omega which image to deploy.

### Athenz Identity

```yaml
requireTLSCerts: true           # Auto-mount Athenz certs to /var/run/athenz/
identityv3: true                # Use Athenz identity v3
```

Omega automatically provisions Athenz certificates and mounts them at:
- `/var/run/athenz/service.cert.pem`
- `/var/run/athenz/service.key.pem`

The service identity is constructed as: `{athensDomain}.{appName}`
- Nonprod: `sre.kafka-consumer-nonprod-k8s.kafka-consumer`
- Prod: `sre.kafka-consumer-k8s.kafka-consumer`

### Resource Allocation

```yaml
resources:
  limits:
    memory:
      nonprod: 2Gi
      prod: 4Gi
  requests:
    cpu:
      nonprod: 500m
      prod: 1
    memory:
      nonprod: 1Gi
      prod: 2Gi
```

Resources are specified separately for nonprod and prod environments.

### Sidecars

```yaml
sidecars:
  splunk:
    instance: deploy.splunk.global.media.yahoo.com:5500
    omegaSplunkClientName: omega_kafka-consumer
```

The Splunk sidecar collects logs from directories specified in `custom.appLogsDirs` and forwards them to Splunk.

### Config Mounts

```yaml
configMounts:
  - name: app-config
    mountPath: /app/config
    source: config              # Maps to deploy_target/omega/config/
```

Files placed in `deploy_target/omega/config/` are mounted into the container at `/app/config/`.

### Persistent Volume Claims (PVC)

```yaml
pvcMounts:
  - name: action-history
    claimName: kafka-consumer-history
    mountPath: /data/automation_consumer
```

PVCs provide persistent storage that survives pod restarts. The action history file is stored here to prevent duplicate action execution.

### Jobs (Deployment Targets)

```yaml
jobs:
  # Development environment
  component:
    environment: development
    colo: use1
    cluster: omega-aws
    clusterEnvironment: centraltech-nonprod1
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-history-dev
        mountPath: /data/automation_consumer

  # Production environment
  deploy-production:
    environment: production
    colo: use1
    cluster: omega-aws
    clusterEnvironment: centraltech-prod1
    pvcMounts:
      - name: action-history
        claimName: kafka-consumer-history-prod
        mountPath: /data/automation_consumer
```

For AWS EKS deployments, you must specify:
- `cluster: omega-aws` - Indicates AWS-based cluster
- `clusterEnvironment` - The specific cluster environment (e.g., `centraltech-nonprod1`)
- `colo` - The AWS region (e.g., `use1` for us-east-1)

The full cluster name is: `{cluster}.{clusterEnvironment}.{colo}`

---

## Omega Scripts

Scripts in `deploy_target/omega/scripts/` are executed at different stages of the container lifecycle.

### app_configure

**When**: During Docker image build
**Purpose**: Set up directories and permissions

```bash
#!/bin/bash
set -ex

# Create runtime directories
mkdir -p /app/config
mkdir -p /app/logs
mkdir -p /data/automation_consumer
mkdir -p /var/run/kafka_consumer

# Set permissions
chmod -R 755 /app
chmod -R 755 /data
```

### app_start

**When**: Container startup
**Purpose**: Wait for Athenz certs and start the application

Key operations:
1. Wait for Athenz certificates (up to 5 minutes)
2. Set environment variables for config paths
3. Start the kafka_consumer process
4. Store PID for graceful shutdown

```bash
# Wait for Athenz certificates
while [ ! -f "$ATHENZ_CERT" ] || [ ! -f "$ATHENZ_KEY" ]; do
    sleep 5
done

# Start consumer
/opt/y/1.0/bin/kafka_consumer \
    --config "$CONFIG_FILE" \
    --log-level "$LOG_LEVEL" &
```

### app_prestop

**When**: Before container termination (pod deletion, scaling down)
**Purpose**: Graceful shutdown of the Kafka consumer

The script:
1. Sends SIGTERM to the consumer process
2. Waits up to 60 seconds for graceful shutdown
3. Force kills if necessary

This ensures the Kafka consumer commits offsets and disconnects cleanly.

---

## Screwdriver Pipeline (screwdriver.yaml)

The Screwdriver pipeline builds and deploys the application.

### Pipeline Flow

```
~pr/~commit → validation → python_package → docker → component → deploy-production
                                                         ↑              ↑
                                                    (auto)         (manual)
```

### Jobs

| Job | Template | Purpose |
|-----|----------|---------|
| `validation` | `python-2504/validate_multiple` | Run tests with tox (py311, py312) |
| `python_package` | `python-2504/package_python` | Build Python wheel |
| `docker` | `python-2504/simple-docker` | Build and push Docker image |
| `component` | `omega/k8s-generic@latest` | Deploy to dev (nonprod) |
| `deploy-production` | `omega/k8s-generic@latest` | Deploy to prod (manual trigger) |

### Omega Deployment Jobs

```yaml
component:
    requires: [docker]
    template: omega/k8s-generic@latest
    steps:
        - copy-config: |
            mkdir -p ${SD_SOURCE_DIR}/deploy_target/omega/config
            cp config/development-config.yaml ${SD_SOURCE_DIR}/deploy_target/omega/config/config.yaml
    environment:
        PUBLISH: true
        DEPLOY_TO_KDNC_K8S: true      # Use KDNC/EKS deployment path
        DEPLOY_TO_OLD_K8S: false       # Disable old k8s deployment
```

Key environment variables:
- `PUBLISH: true` - Build and push the Docker image
- `DEPLOY_TO_KDNC_K8S: true` - Deploy to new EKS clusters (uses `k8s-ci app init-kdnc` and `k8s-ci app deploy-kdnc`)
- `DEPLOY_TO_OLD_K8S: false` - Skip old Kubernetes clusters

The `copy-config` step copies the environment-specific config file to `deploy_target/omega/config/config.yaml`, which gets mounted into the container via `configMounts`.

---

## Persistent Volume Claims (PVC)

PVCs must be created before deployment. The kafka-consumer uses PVCs to persist action history across pod restarts.

### Why PVC is Needed

The consumer tracks which actions have been executed in a JSON file (`automation_action_history.json`). Without persistent storage:
- Pod restarts would lose action history
- Duplicate actions could be executed

### PVC Specifications

**Nonprod PVC** (`pvc-nonprod.yaml`):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-consumer-history-dev
  namespace: sre.kafka-consumer-nonprod-k8s
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-default
  resources:
    requests:
      storage: 1Gi
```

**Prod PVC** (`pvc-prod.yaml`):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-consumer-history-prod
  namespace: sre.kafka-consumer-prod-k8s
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-default
  resources:
    requests:
      storage: 1Gi
```

### Storage Class

AWS EKS clusters in use1 have `efs-default` storage class enabled, which provides:
- EFS (Elastic File System) backed storage
- `ReadWriteMany` access mode (multiple pods can mount simultaneously)

### Creating PVCs

```bash
# Create nonprod PVC
kubectl --context omega-aws.centraltech-nonprod1.use1 \
  -n sre.kafka-consumer-nonprod-k8s \
  apply -f pvc-nonprod.yaml

# Create prod PVC
kubectl --context omega-aws.centraltech-prod1.use1 \
  -n sre.kafka-consumer-prod-k8s \
  apply -f pvc-prod.yaml

# Verify PVCs
kubectl --context omega-aws.centraltech-nonprod1.use1 \
  -n sre.kafka-consumer-nonprod-k8s \
  get pvc
```

---

## Deployment Workflow

### First-Time Setup

1. **Create PVCs** in both nonprod and prod clusters (see above)
2. **Verify Athenz/IAM** setup via Terraform (`sre-terraform-kafka-consumer-eks`)
3. **Merge to main** to trigger the pipeline

### Regular Deployments

1. **Push code** to branch or create PR
2. **Validation** runs automatically on PR/commit
3. **Dev deployment** (`component` job) runs after docker build succeeds
4. **Prod deployment** (`deploy-production`) requires manual trigger after dev success

### Manual Trigger for Production

Production deployment requires manual approval. In Screwdriver UI:
1. Navigate to the pipeline
2. Click on `deploy-production` job
3. Click "Start" to trigger deployment

---

## Troubleshooting

### Check Pod Status
```bash
kubectl --context omega-aws.centraltech-nonprod1.use1 \
  -n sre.kafka-consumer-nonprod-k8s \
  get pods
```

### View Pod Logs
```bash
kubectl --context omega-aws.centraltech-nonprod1.use1 \
  -n sre.kafka-consumer-nonprod-k8s \
  logs <pod-name> -c app
```

### Check Athenz Certificates
```bash
kubectl --context omega-aws.centraltech-nonprod1.use1 \
  -n sre.kafka-consumer-nonprod-k8s \
  exec <pod-name> -c app -- ls -la /var/run/athenz/
```

### View PVC Status
```bash
kubectl --context omega-aws.centraltech-nonprod1.use1 \
  -n sre.kafka-consumer-nonprod-k8s \
  describe pvc kafka-consumer-history-dev
```

---

---

## CKMS Secret Access

The application fetches secrets from CKMS at runtime using Athenz certificates.

### How It Works

```
Pod starts → Omega provisions Athenz certs → App calls SecretManager → CKMS returns secrets
```

1. **Omega provisions identity**: When the pod starts, Omega's identity sidecar creates Athenz certificates for the service identity (e.g., `sre.kafka-consumer-nonprod-k8s.kafka-consumer`)

2. **App reads config**: The config file references secrets by name (e.g., `bigpanda-prod-scram-key`)

3. **SecretManager fetches secrets**: Using the Athenz certs, the app calls CKMS API to get the actual secret values

4. **CKMS checks access**: CKMS verifies the service identity is in the access role before returning secrets

### CKMS Key Group Setup

Our secrets are stored in the `bigpanda.keys.sre` key group, which is managed by the `sre.tooling` Athenz domain.

**Access is controlled by the role:**
```
sre.tooling:role.paranoids.ppse.ckms.ykeykey_aws.res_group.bigpanda.keys.sre.access
```

### Granting Access to New Services

The admin of `sre.tooling` domain must add the service identities to the CKMS access role:

```bash
# For NONPROD
zms-cli -d sre.tooling add-role-member \
  paranoids.ppse.ckms.ykeykey_aws.res_group.bigpanda.keys.sre.access \
  sre.kafka-consumer-nonprod-k8s.kafka-consumer

# For PROD
zms-cli -d sre.tooling add-role-member \
  paranoids.ppse.ckms.ykeykey_aws.res_group.bigpanda.keys.sre.access \
  sre.kafka-consumer-k8s.kafka-consumer
```

### Verifying Access

To check if a service has access:
```bash
zms-cli -d sre.tooling show-role \
  paranoids.ppse.ckms.ykeykey_aws.res_group.bigpanda.keys.sre.access
```

---

## Related Documentation

- [Omega Template Reference](docs/omega%20template.md)
- [Application Config Files](config/)
- [Terraform EKS Setup](../sre-terraform-kafka-consumer-eks)
