# AWS EKS Deployment Plan for Kafka Consumer

## Executive Summary

Deploy the `yahoo.kafka_consumer` application to AWS EKS cluster (`omega-aws_centraltech-nonprod1_use1`) for the first time. This is a greenfield deployment using IAM roles (IRSA) for authentication and standard Kubernetes manifests for orchestration.

**Timeline:** 2-3 weeks including validation
**Risk Level:** Low (new deployment, no existing systems affected)
**Deployment Type:** Greenfield (not a migration)

---

## Key Design Decisions

### 1. Terraform Repository: Create New sre-terraform-kafka-consumer Repo

**Decision:** Create dedicated terraform repository for kafka-consumer infrastructure

**Rationale:**
- Clean separation of concerns (rootly schedules vs kafka-consumer infrastructure)
- Independent versioning and release cycles
- Easier to manage permissions and access control
- Follows pattern: one infrastructure repo per major application

**Structure:**
```
sre-terraform-kafka-consumer/
├── terraform/
│   ├── modules/
│   │   └── kafka-consumer-service/
│   │       ├── main.tf
│   │       ├── athenz.tf
│   │       ├── iam.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
│   ├── environments/
│   │   ├── development/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend-config.hcl
│   │   └── production/
│   │       ├── main.tf
│   │       ├── terraform.tfvars
│   │       └── backend-config.hcl
│   ├── backend.tf
│   └── versions.tf
├── README.md
└── .gitignore
```

### 2. Secrets Management: Use CKMS with "aws" Environment

**Decision:** Use CKMS "aws" environment with IAM authentication

**Rationale:**
- CKMS "aws" environment specifically designed for AWS EKS workloads
- Keys accessible from anywhere (not datacenter-specific)
- Consistent with Yahoo infrastructure patterns
- Integrates with IRSA (IAM Roles for Service Accounts)
- Application code (`secret_manager.py`) needs updates to support IAM auth

**Implementation:**
- Update `SecretManager` class to support IAM authentication mode
- IRSA provides IAM credentials automatically to pod
- Add `use_iam_auth: true` and `ckms_environment: aws` to config files
- CKMS client library should auto-detect IAM credentials from environment

### 3. Config Management: Use ConfigMaps for Non-Secret Configuration

**Decision:** Use Kubernetes ConfigMaps for application configuration

**Rationale:**
- Native Kubernetes pattern for configuration management
- Easy to update without rebuilding Docker images
- Version-controlled and templated per environment (dev/prod)
- Secrets remain in CKMS (not in ConfigMaps)
- Supports Kustomize overlays for environment-specific configs

**Implementation:**
- Base config in `k8s/base/configmap.yaml`
- Environment-specific overrides in `k8s/overlays/{dev,prod}/configmap.yaml`
- Mount ConfigMap as file at `/app/config/config.yaml`
- Application reads config file at startup

---

## Target Architecture

### AWS EKS Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS EKS CLUSTER                              │
│  omega-aws_centraltech-nonprod1_use1                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Namespace: sre.kafka-consumer-k8s                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ServiceAccount: kafka-consumer                         │   │
│  │  (IRSA annotation: IAM role ARN)                        │   │
│  │         ↓                                                │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │  Pod: kafka-consumer                             │   │   │
│  │  │  ┌────────────────────────────────────────────┐  │   │   │
│  │  │  │ Container: kafka-consumer                  │  │   │   │
│  │  │  │ - IAM credentials via IRSA                 │  │   │   │
│  │  │  │ - Connects to MSK (SCRAM or IAM)           │  │   │   │
│  │  │  │ - Fetches secrets from CKMS (IAM auth)     │  │   │   │
│  │  │  │ - Writes action history to EBS PVC         │  │   │   │
│  │  │  └────────────────────────────────────────────┘  │   │   │
│  │  │                                                   │   │   │
│  │  │  Volumes:                                         │   │   │
│  │  │  - ConfigMap: /app/config/config.yaml            │   │   │
│  │  │  - PVC: /data/automation_consumer (EBS gp3)      │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                        ↓                     ↓
                ┌───────────────┐    ┌────────────────┐
                │  CKMS (aws)   │    │  MSK Cluster   │
                │  IAM Auth     │    │  SCRAM/IAM     │
                └───────────────┘    └────────────────┘
```

### Key Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Authentication** | IAM (IRSA) | Pod identity and CKMS access |
| **Secrets** | CKMS (aws environment) | Kafka credentials, API keys |
| **Storage** | EBS gp3 PVC | Persistent action history |
| **Configuration** | ConfigMap | Application config (non-secrets) |
| **Deployment** | Kubernetes manifests | Pod orchestration |
| **Infrastructure** | Terraform | IAM roles, Athenz config |

---

## Implementation Plan

### Phase 1: Terraform Infrastructure Setup (Week 1)

#### Files to Create:

1. **New Terraform Repository: sre-terraform-kafka-consumer**
   ```
   sre-terraform-kafka-consumer/
   ├── terraform/
   │   ├── modules/
   │   │   └── kafka-consumer-service/
   │   │       ├── main.tf          # Module orchestration
   │   │       ├── athenz.tf        # Athenz services, groups, roles, policies
   │   │       ├── iam.tf           # IAM roles via tf-omega-eks-identity
   │   │       ├── variables.tf     # Module inputs
   │   │       └── outputs.tf       # IAM role ARN, etc.
   │   ├── environments/
   │   │   ├── development/
   │   │   │   ├── main.tf              # Dev-specific config
   │   │   │   ├── terraform.tfvars     # Dev values
   │   │   │   └── backend-config.hcl   # S3 state config
   │   │   └── production/
   │   │       ├── main.tf              # Prod-specific config
   │   │       ├── terraform.tfvars     # Prod values
   │   │       └── backend-config.hcl   # S3 state config
   │   ├── backend.tf           # S3 backend definition
   │   └── versions.tf          # Terraform & provider versions
   ├── README.md                # Documentation
   ├── .gitignore
   └── screwdriver.yaml         # CI/CD for terraform apply
   ```

2. **Key Terraform Resources:**
   - Athenz services: `kafka-consumer-development`, `kafka-consumer-production`
   - Athenz groups: `omega.eksapi.access`, `omega.splunk.access`
   - Athenz roles: `k8s_developer` (team members), `k8s_deployer` (Screwdriver jobs)
   - IAM roles via `tf-omega-eks-identity` module (IRSA)
   - IAM policies: CKMS access, MSK access (if using IAM auth)

3. **Important Variables:**
   ```hcl
   athenz_domain        = "sre.kafka-consumer-k8s"
   service_name         = "kafka-consumer-development" or "kafka-consumer-production"
   cluster_name         = "omega-aws_centraltech-nonprod1_use1"
   namespace            = "sre.kafka-consumer-k8s"
   aws_account_id       = "YOUR_AWS_ACCOUNT_ID"
   consumer_group_id    = "sre-kafka-consumer-dev-eks" or "sre-kafka-consumer-prod-eks"
   msk_auth_method      = "scram"  # or "iam"
   ckms_kms_key_arn     = "arn:aws:kms:us-east-1:ACCOUNT:key/CKMS_KEY_ID"
   ```

4. **Backend Configuration:**
   Create new S3 bucket and DynamoDB table for terraform state:

   **Option A - Use existing SRE team S3 bucket (if available):**
   ```hcl
   bucket         = "sre-terraform-state"
   key            = "kafka-consumer/development/terraform.tfstate"
   region         = "us-east-1"
   dynamodb_table = "sre-terraform-locks"
   encrypt        = true
   ```

   **Option B - Create new dedicated infrastructure:**
   1. First, bootstrap S3 bucket and DynamoDB table:
      ```bash
      cd sre-terraform-kafka-consumer/terraform/bootstrap
      terraform init
      terraform apply  # Creates bucket and table
      ```
   2. Then use for main terraform:
      ```hcl
      bucket         = "sre-kafka-consumer-terraform-state"
      key            = "kafka-consumer/development/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "sre-kafka-consumer-terraform-locks"
      encrypt        = true
      ```

#### Steps:

1. Create terraform directory structure in `sre-terraform-rootly`
2. Write terraform modules and environment configs
3. Initialize and plan:
   ```bash
   cd terraform/kafka-consumer/environments/development
   terraform init -backend-config=backend-config.hcl
   terraform plan
   ```
4. Apply to create infrastructure:
   ```bash
   terraform apply
   ```
5. Save IAM role ARN output for later use

---

### Phase 2: Kubernetes Manifests (Week 1)

#### Files to Create:

```
k8s/
├── base/
│   ├── namespace.yaml          # sre.kafka-consumer-k8s namespace
│   ├── serviceaccount.yaml     # ServiceAccount with IRSA annotation
│   ├── deployment.yaml         # Deployment spec
│   ├── pvc.yaml                # PersistentVolumeClaim (10Gi EBS gp3)
│   └── configmap.yaml          # Base config template
└── overlays/
    ├── development/
    │   ├── kustomization.yaml  # Kustomize overlay
    │   ├── configmap.yaml      # Dev-specific config values
    │   └── deployment-patch.yaml  # Dev resource limits
    └── production/
        ├── kustomization.yaml  # Kustomize overlay
        ├── configmap.yaml      # Prod-specific config values
        └── deployment-patch.yaml  # Prod resource limits
```

#### Key Manifest Features:

**ServiceAccount:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka-consumer
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT:role/kafka-consumer-development"
```

**Deployment:**
- Single replica (`replicas: 1`) - required due to file-based state
- Strategy: `Recreate` (avoid multiple pods accessing same PVC)
- Security context: Run as non-root user (UID 1000)
- Environment variables:
  - `AWS_REGION=us-east-1`
  - `AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
  - `AWS_ROLE_ARN=<IAM_ROLE_ARN>`
- Resources:
  - Requests: 500m CPU, 1Gi memory
  - Limits: 1000m CPU, 2Gi memory
- Liveness probe: Check PID file exists and process alive
- Readiness probe: Check process alive and uptime >= 30s
- Grace period: 120s (allow Kafka offset commits)

**PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-consumer-history
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests:
      storage: 10Gi
```

**ConfigMap Structure:**
```yaml
data:
  config.yaml: |
    global:
      aws_shared_credentials_file: /tmp/awscreds
      logfile: /app/logs/consumer.log

    athenz:
      certificate: /var/run/secrets/irsa/token  # Placeholder
      key: /var/run/secrets/irsa/token          # Placeholder
      use_iam_auth: true  # NEW FLAG for IAM authentication

    secrets:
      - name: bigpanda-prod-scram-key
        ckms_key_group: bigpanda.keys.sre
        ckms_key_name: bigpanda.prod.msk.sre
        ckms_environment: aws  # NEW FIELD - specifies aws environment

    kafka:
      brokers: [broker1:443, broker2:443, broker3:443]
      consumer_group_id: sre-kafka-consumer-dev-eks  # DIFFERENT per environment
      topics: [sre]

    # ... rest of config (BigPanda, actions, incident_conditions)
```

---

### Phase 3: Application Code Changes (Week 1)

#### Files to Modify:

**1. src/yahoo/kafka_consumer/secret_manager.py**

Add IAM authentication support for CKMS:

```python
class SecretManager:
    def __init__(self, config: dict):
        self.config = config
        self.secrets = config["secrets"]

        # Check if running in EKS with IAM auth
        self.use_iam_auth = config.get('athenz', {}).get('use_iam_auth', False)

        if not self.use_iam_auth:
            # Traditional Athenz certificate authentication
            self.cert = config['athenz']['certificate']
            self.key = config['athenz']['key']
            logging.info("Using Athenz x509 certificate authentication")
        else:
            # IAM authentication via IRSA
            self.cert = None
            self.key = None
            logging.info("Using IAM authentication for CKMS (EKS IRSA)")

    def get_ckms_key(self, x509_cert: str, x509_key: str, ckms_key: str,
                     ckms_key_group: str, ckms_environment: str = None):
        ckms_client = YSecureOathCKMS()

        if self.use_iam_auth:
            # Use IAM authentication - CKMS client automatically detects
            # IAM credentials from AWS_WEB_IDENTITY_TOKEN_FILE and AWS_ROLE_ARN
            logging.debug("Fetching CKMS key using IAM authentication")
            if ckms_environment:
                ckms_client.environment = ckms_environment
        else:
            # Use Athenz x509 certificate authentication
            ckms_client.cert_filename = x509_cert
            ckms_client.key_filename = x509_key

        return ckms_client.get_key(ckms_key, ckms_key_group)
```

**Changes:**
- Add `use_iam_auth` flag detection from config
- Add `ckms_environment` parameter support
- Auto-detect IAM credentials from environment variables set by IRSA

**2. deploy_target/eks/scripts/app_start**

Create new startup script for EKS (no Athenz cert wait):

```bash
#!/bin/bash
set -ex

echo "=== Starting kafka_consumer in EKS ==="

# Set configuration paths
export CONFIG_FILE="${CONFIG_FILE:-/app/config/config.yaml}"
export LOG_LEVEL="${LOG_LEVEL:-INFO}"
export ACTION_HISTORY_FILE="/data/automation_consumer/automation_action_history.json"

# Verify config exists
if [ ! -f "$CONFIG_FILE" ]; then
    echo "ERROR: Config file not found at $CONFIG_FILE"
    exit 1
fi

# Log IAM role info (for debugging)
echo "IAM Role: $AWS_ROLE_ARN"
echo "ServiceAccount Token: $AWS_WEB_IDENTITY_TOKEN_FILE"

# PID file for health checks
PIDFILE="/var/run/kafka_consumer/consumer.pid"

# Start the consumer
/opt/y/1.0/bin/kafka_consumer \
    --config "$CONFIG_FILE" \
    --log-level "$LOG_LEVEL" &

echo $! > "$PIDFILE"
echo "Started with PID $(cat $PIDFILE)"

# Keep container running
wait
```

**Key Changes:**
- Removed Athenz certificate wait loop (lines 8-22 from original)
- Removed certificate validation checks
- Immediate startup after config validation

**3. Dockerfile**

Update to support EKS deployment:

```dockerfile
# ... existing base image and python setup ...

# Create necessary directories
RUN mkdir -p /app/logs /app/config /var/run/kafka_consumer /data/automation_consumer && \
    chmod 755 /app/logs /app/config /var/run/kafka_consumer /data/automation_consumer

# Copy EKS startup scripts
COPY deploy_target/eks/scripts/app_start /usr/local/bin/app_start
COPY deploy_target/eks/scripts/app_prestop /usr/local/bin/app_prestop
RUN chmod +x /usr/local/bin/app_start /usr/local/bin/app_prestop

# Run as non-root user
USER 1000

CMD ["/usr/local/bin/app_start"]
```

**4. config/development-config.yaml & config/production-config.yaml**

Update to reference CKMS "aws" environment:

```yaml
athenz:
  certificate: /var/run/secrets/irsa/token  # Placeholder - not used in EKS
  key: /var/run/secrets/irsa/token          # Placeholder - not used in EKS
  use_iam_auth: true  # Enable IAM authentication

secrets:
  - name: bigpanda-prod-scram-key
    ckms_key_group: bigpanda.keys.sre
    ckms_key_name: bigpanda.prod.msk.sre
    ckms_environment: aws  # NEW - specify aws environment
  # ... update all other secrets similarly

kafka:
  brokers: [broker1:443, broker2:443, broker3:443]
  consumer_group_id: sre-kafka-consumer-dev-eks  # For dev config
  # consumer_group_id: sre-kafka-consumer-prod-eks  # For prod config
  topics: [sre]
```

---

### Phase 4: CI/CD Pipeline Updates (Week 1-2)

#### Files to Modify:

**screwdriver.yaml**

Add new jobs for EKS deployment:

```yaml
version: 4

shared:
    environment:
        TOX_ENVLIST: py311,py312
        EKS_CLUSTER_NAME: omega-aws_centraltech-nonprod1_use1
        EKS_NAMESPACE: sre.kafka-consumer-k8s

jobs:
    # Existing jobs (unchanged)
    validation:
        template: python-2504/validate_multiple
        requires: [~pr, ~commit]

    python_package:
        template: python-2504/package_python
        requires: [validation]

    docker:
        template: python-2504/simple-docker
        requires: [python_package]

    # NEW: Terraform plan for development
    # Note: This assumes sre-terraform-kafka-consumer repo is checked out
    # or available via git submodule or external trigger
    terraform-plan-dev:
        requires: [docker]
        image: hashicorp/terraform:1.5
        steps:
            - checkout-terraform: |
                # Clone terraform repo
                git clone git@git.ouryahoo.com:sre/sre-terraform-kafka-consumer.git
                cd sre-terraform-kafka-consumer/terraform/environments/development
            - init-and-plan: |
                cd sre-terraform-kafka-consumer/terraform/environments/development
                terraform init -backend-config=backend-config.hcl
                terraform plan -out=tfplan

    # NEW: Terraform apply for development
    terraform-apply-dev:
        requires: [terraform-plan-dev]
        image: hashicorp/terraform:1.5
        steps:
            - checkout-terraform: |
                git clone git@git.ouryahoo.com:sre/sre-terraform-kafka-consumer.git
            - apply: |
                cd sre-terraform-kafka-consumer/terraform/environments/development
                terraform init -backend-config=backend-config.hcl
                terraform apply -auto-approve

    # NEW: Deploy to EKS development
    deploy-eks-dev:
        requires: [terraform-apply-dev]
        image: docker.ouroath.com:4443/omega/kubectl:latest
        environment:
            VERSION: ${SD_BUILD_ID}
        steps:
            - setup-kubectl: |
                aws eks update-kubeconfig \
                    --region us-east-1 \
                    --name ${EKS_CLUSTER_NAME}
                kubectl get nodes

            - get-iam-role: |
                git clone git@git.ouryahoo.com:sre/sre-terraform-kafka-consumer.git
                cd sre-terraform-kafka-consumer/terraform/environments/development
                terraform init -backend-config=backend-config.hcl
                IAM_ROLE_ARN=$(terraform output -raw irsa_role_arn)
                echo "$IAM_ROLE_ARN" > /tmp/iam_role_arn

            - deploy: |
                # Update ServiceAccount annotation
                IAM_ROLE_ARN=$(cat /tmp/iam_role_arn)
                kubectl annotate serviceaccount kafka-consumer \
                    -n ${EKS_NAMESPACE} \
                    eks.amazonaws.com/role-arn="$IAM_ROLE_ARN" \
                    --overwrite

                # Apply kustomize with version
                cd k8s/overlays/development
                kustomize edit set image \
                    docker.ouroath.com:4443/yahoo/kafka_consumer:${VERSION}
                kubectl apply -k .

                # Wait for rollout
                kubectl rollout status deployment/kafka-consumer \
                    -n ${EKS_NAMESPACE} \
                    --timeout=10m

    # Production Terraform and deploy (manual trigger)
    terraform-plan-prod:
        requires: [~sd@1132405:deploy-eks-dev]
        image: hashicorp/terraform:1.5
        steps:
            - checkout-terraform: |
                git clone git@git.ouryahoo.com:sre/sre-terraform-kafka-consumer.git
            - init-and-plan: |
                cd sre-terraform-kafka-consumer/terraform/environments/production
                terraform init -backend-config=backend-config.hcl
                terraform plan -out=tfplan

    terraform-apply-prod:
        requires: [~sd@1132405:terraform-plan-prod]
        image: hashicorp/terraform:1.5
        steps:
            - checkout-terraform: |
                git clone git@git.ouryahoo.com:sre/sre-terraform-kafka-consumer.git
            - apply: |
                cd sre-terraform-kafka-consumer/terraform/environments/production
                terraform init -backend-config=backend-config.hcl
                terraform apply -auto-approve

    deploy-eks-prod:
        requires: [terraform-apply-prod]
        image: docker.ouroath.com:4443/omega/kubectl:latest
        environment:
            VERSION: ${SD_BUILD_ID}
        steps:
            # Similar to deploy-eks-dev but with production overlay
            - setup-kubectl: |
                aws eks update-kubeconfig \
                    --region us-east-1 \
                    --name ${EKS_CLUSTER_NAME}

            - get-iam-role: |
                git clone git@git.ouryahoo.com:sre/sre-terraform-kafka-consumer.git
                cd sre-terraform-kafka-consumer/terraform/environments/production
                terraform init -backend-config=backend-config.hcl
                IAM_ROLE_ARN=$(terraform output -raw irsa_role_arn)
                echo "$IAM_ROLE_ARN" > /tmp/iam_role_arn

            - deploy: |
                IAM_ROLE_ARN=$(cat /tmp/iam_role_arn)
                kubectl annotate serviceaccount kafka-consumer-prod \
                    -n ${EKS_NAMESPACE} \
                    eks.amazonaws.com/role-arn="$IAM_ROLE_ARN" \
                    --overwrite

                cd k8s/overlays/production
                kustomize edit set image \
                    docker.ouroath.com:4443/yahoo/kafka_consumer:${VERSION}
                kubectl apply -k .
                kubectl rollout status deployment/kafka-consumer \
                    -n ${EKS_NAMESPACE} \
                    --timeout=10m
```

**Key Pipeline Changes:**
- Add Terraform plan/apply jobs before EKS deployment
- Add EKS deployment jobs using kubectl + kustomize
- Use manual triggers (`~sd@...`) for production deployments

---

### Phase 5: Documentation (Week 2)

#### Files to Create:

**1. docs/eks-deployment-guide.md**
- Architecture overview with diagrams
- Step-by-step deployment instructions
- Verification procedures
- Troubleshooting guide
- Operations guide (scaling, upgrades, logs)

**2. docs/eks-operations-runbook.md**
- Pre-deployment checklist
- Deployment timeline
- Success criteria and metrics
- Contact information

**3. Update docs/screwdriver-deployment-guide.md**
- Add EKS deployment section
- Document differences from potential Omega deployment
- Update pipeline flow diagrams

---

## Deployment Timeline

### Week 1: Infrastructure Setup and Code Preparation
- [ ] Create new git repository: `sre-terraform-kafka-consumer`
- [ ] Set up Terraform bootstrap (S3 bucket, DynamoDB table)
- [ ] Create Terraform infrastructure modules
- [ ] Deploy Terraform to development environment
- [ ] Create Kubernetes manifests (base + overlays)
- [ ] Update application code (SecretManager, startup scripts, Dockerfile)
- [ ] Update CI/CD pipeline in kafka_consumer repo (screwdriver.yaml)
- [ ] Create documentation

### Week 2: Development Deployment and Validation
- [ ] Deploy infrastructure via Terraform
- [ ] Deploy application to EKS development via Screwdriver
- [ ] Verify pod starts successfully
- [ ] Verify CKMS secrets accessible via IAM
- [ ] Verify Kafka connectivity (MSK cluster)
- [ ] Test incident processing end-to-end
- [ ] Validate action execution (Screwdriver, Slack, webhooks)
- [ ] Monitor logs and metrics for 3-5 days

### Week 3: Production Deployment and Validation
- [ ] Deploy Terraform to production environment
- [ ] Deploy application to EKS production via Screwdriver
- [ ] Verify production deployment successful
- [ ] Monitor production consumer for 1 week
- [ ] Validate incident processing in production
- [ ] Set up monitoring and alerting
- [ ] Document operational procedures

### Week 4: Handoff and Documentation (Optional)
- [ ] Team training on EKS operations
- [ ] Finalize runbooks and troubleshooting guides
- [ ] Set up on-call procedures
- [ ] Conduct knowledge transfer
- [ ] Post-deployment review

---

## Verification Steps

### Development Environment

**1. Infrastructure Verification:**
```bash
# Verify Terraform state
cd sre-terraform-kafka-consumer/terraform/environments/development
terraform state list

# Check IAM role exists
aws iam get-role --role-name kafka-consumer-development

# Verify IRSA trust policy
aws iam get-role --role-name kafka-consumer-development \
    --query 'Role.AssumeRolePolicyDocument'
```

**2. Kubernetes Resource Verification:**
```bash
# Verify namespace
kubectl get ns sre.kafka-consumer-k8s

# Verify ServiceAccount has IAM annotation
kubectl get sa kafka-consumer -n sre.kafka-consumer-k8s -o yaml \
    | grep eks.amazonaws.com/role-arn

# Verify PVC is bound
kubectl get pvc -n sre.kafka-consumer-k8s

# Check pod status
kubectl get pods -n sre.kafka-consumer-k8s -l app=kafka-consumer
```

**3. Application Verification:**
```bash
# Check logs for successful startup
POD=$(kubectl get pods -n sre.kafka-consumer-k8s -l app=kafka-consumer -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n sre.kafka-consumer-k8s $POD --tail=100

# Expected log entries:
# - "Using IAM authentication for CKMS (EKS IRSA)"
# - "Kafka authentication method selected: scram"
# - "Subscribed to topic"

# Verify CKMS access
kubectl exec -n sre.kafka-consumer-k8s $POD -- \
    python3 -c "from yahoo.kafka_consumer.secret_manager import SecretManager; print('CKMS OK')"

# Check action history file exists
kubectl exec -n sre.kafka-consumer-k8s $POD -- \
    ls -lh /data/automation_consumer/automation_action_history.json
```

**4. End-to-End Functional Test:**
- Trigger a test incident in BigPanda
- Verify consumer processes the incident
- Verify actions execute successfully
- Check action history is updated
- Verify Slack notifications sent

### Production Environment

**Same verification steps as development, but:**
- Use production terraform environment
- Use production namespace and resources
- Monitor for production traffic patterns

---

## Troubleshooting and Recovery

### Emergency Stop (2 minutes)

If critical issues are discovered and you need to stop the consumer immediately:

```bash
# Stop EKS consumer
kubectl scale deployment kafka-consumer -n sre.kafka-consumer-k8s --replicas=0

# Notify team via Slack
```

### Recovery and Restart (10 minutes)

To restart the consumer after fixing issues:

```bash
# Option 1: Redeploy from Screwdriver
# Trigger the deploy-eks-dev or deploy-eks-prod job

# Option 2: Manual restart
kubectl scale deployment kafka-consumer -n sre.kafka-consumer-k8s --replicas=1

# Verify restart
kubectl get pods -n sre.kafka-consumer-k8s -l app=kafka-consumer
kubectl logs -f -n sre.kafka-consumer-k8s deployment/kafka-consumer
```

### Complete Redeployment (30 minutes)

If the deployment is severely broken and needs to be completely redeployed:

1. Delete existing resources:
   ```bash
   kubectl delete deployment kafka-consumer -n sre.kafka-consumer-k8s
   kubectl delete configmap kafka-consumer-config -n sre.kafka-consumer-k8s
   # Note: Do NOT delete PVC unless action history needs to be reset
   ```

2. Redeploy via Screwdriver or manually:
   ```bash
   cd k8s/overlays/development  # or production
   kubectl apply -k .
   ```

3. Monitor deployment:
   ```bash
   kubectl rollout status deployment/kafka-consumer -n sre.kafka-consumer-k8s
   ```

---

## Success Criteria

Deployment considered successful when:
- [ ] Terraform infrastructure deployed successfully (dev and prod)
- [ ] EKS deployment stable for 1 week in development
- [ ] EKS deployment stable for 1 week in production
- [ ] CKMS authentication working reliably (< 0.1% failure rate)
- [ ] Kafka consumer processing incidents correctly
- [ ] Action history persisting correctly to PVC
- [ ] All action types execute successfully (Screwdriver, Slack, webhooks, etc.)
- [ ] No critical errors in logs
- [ ] Team comfortable with EKS operations
- [ ] All monitoring and alerting functional
- [ ] Documentation complete and validated
- [ ] Runbooks tested and operational

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| **CKMS auth fails in EKS** | Test thoroughly in dev; CKMS team support available |
| **IAM role trust policy misconfigured** | Validate IRSA setup; use terraform module examples |
| **PVC data loss** | EBS volumes are durable; enable EBS snapshots for backups |
| **Kafka connection issues** | Verify security groups allow EKS → MSK traffic; test connectivity |
| **Action history corruption** | Can initialize empty (no existing data to lose) |
| **Secrets not accessible** | Validate CKMS key permissions in Athenz before deployment |
| **Team unfamiliar with EKS ops** | Comprehensive docs + training; start with dev environment |
| **Terraform state corruption** | Use S3 versioning and DynamoDB locking |

---

## Key Technical Considerations

### 1. CKMS Authentication in EKS

The `ouroath.ckms` library needs to support IAM authentication. Verify:
- Library version supports IAM auth
- Environment variables are correctly set by IRSA
- Trust policy includes correct EKS OIDC provider

If CKMS library doesn't support IAM natively, may need to:
- Fetch temporary credentials via AWS STS AssumeRoleWithWebIdentity
- Pass credentials to CKMS client

### 2. File-Based State Limitation

The application uses `FileBackedDict` for action history, which:
- Requires single replica (cannot scale horizontally)
- Uses `Recreate` deployment strategy
- Requires ReadWriteOnce PVC (EBS, not EFS)

Future enhancement: Migrate to database (DynamoDB, PostgreSQL) for multi-replica support.

### 3. MSK Authentication Options

Currently using SCRAM, but EKS also supports MSK IAM authentication:
- SCRAM: Use CKMS for password, works with current code
- IAM: Use IRSA role, requires `msk_auth.auth_method: iam` in config

Recommend starting with SCRAM (less change), consider IAM later.

### 4. Health Check Probes

Existing health checks use PID file approach:
- Works in EKS without modification
- Liveness: Check process exists
- Readiness: Check process exists + uptime > 30s

Consider adding application-level health endpoint in future.

---

## Critical Files Summary

### Files to Create (New):

**New Git Repository: sre-terraform-kafka-consumer**
- `README.md`
- `.gitignore`
- `screwdriver.yaml` (for terraform CI/CD)

**Terraform (in sre-terraform-kafka-consumer):**
- `terraform/modules/kafka-consumer-service/main.tf`
- `terraform/modules/kafka-consumer-service/athenz.tf`
- `terraform/modules/kafka-consumer-service/iam.tf`
- `terraform/modules/kafka-consumer-service/variables.tf`
- `terraform/modules/kafka-consumer-service/outputs.tf`
- `terraform/environments/development/main.tf`
- `terraform/environments/development/terraform.tfvars`
- `terraform/environments/development/backend-config.hcl`
- `terraform/environments/production/main.tf`
- `terraform/environments/production/terraform.tfvars`
- `terraform/environments/production/backend-config.hcl`
- `terraform/backend.tf`
- `terraform/versions.tf`
- `terraform/bootstrap/main.tf` (optional - if creating new S3 bucket)

**Kubernetes (in kafka_consumer repo):**
- `k8s/base/namespace.yaml`
- `k8s/base/serviceaccount.yaml`
- `k8s/base/deployment.yaml`
- `k8s/base/pvc.yaml`
- `k8s/base/configmap.yaml`
- `k8s/overlays/development/kustomization.yaml`
- `k8s/overlays/development/configmap.yaml`
- `k8s/overlays/development/deployment-patch.yaml`
- `k8s/overlays/production/kustomization.yaml`
- `k8s/overlays/production/configmap.yaml`
- `k8s/overlays/production/deployment-patch.yaml`

**Scripts (in kafka_consumer repo):**
- `deploy_target/eks/scripts/app_start`
- `deploy_target/eks/scripts/app_prestop`

**Documentation (in kafka_consumer repo):**
- `docs/eks-deployment-guide.md`
- `docs/eks-operations-runbook.md`

### Files to Modify (Existing):

**Application Code:**
- `src/yahoo/kafka_consumer/secret_manager.py` - Add IAM authentication support
- `Dockerfile` - Copy EKS scripts, set up directories

**Configuration:**
- `config/development-config.yaml` - Add `use_iam_auth: true`, `ckms_environment: aws`
- `config/production-config.yaml` - Same as development

**CI/CD:**
- `screwdriver.yaml` - Add Terraform and EKS deployment jobs

**Documentation:**
- `docs/screwdriver-deployment-guide.md` - Add EKS deployment section

---

## Questions to Answer Before Implementation

Before proceeding with implementation, please provide answers to these questions:

### 1. AWS Account Details
- What is your AWS account ID?
- What is your S3 bucket name for Terraform state? (Or should we create a new one?)
- What is your DynamoDB table name for Terraform locking? (Or should we create a new one?)

### 2. CKMS Configuration
- Do you have access to CKMS "aws" environment keys?
- Do you know the CKMS KMS key ARN for encryption?
- Have CKMS key permissions been granted to your Athenz domain (`sre.kafka-consumer-k8s`)?

### 3. Team Access
- Who are the team members to add to `k8s_developer` role in Athenz?
- Do you have EKS cluster access configured locally (kubectl)?
- Who should have `k8s_deployer` role for Screwdriver?

### 4. Deployment Preferences
- Should we start with development environment only?
- What is the preferred timeline for production deployment?
- Are there any specific maintenance windows or blackout periods?

### 5. Testing and Validation
- Do you have a test incident we can use for end-to-end validation?
- Who should be notified during testing (Slack channel)?
- What are the success criteria specific to your team?

### 6. Additional Clarifications
- Are there any existing EKS deployments in your team we can reference?
- Do you need training on EKS operations before deployment?
- Are there any company-specific policies or procedures we should follow?

---

## Next Steps

Once this plan is approved and questions are answered:

1. **Week 1:** Create terraform repository and implement infrastructure
2. **Week 1:** Create Kubernetes manifests and update application code
3. **Week 1:** Update CI/CD pipeline in kafka_consumer repo
4. **Week 2:** Deploy to development and validate thoroughly
5. **Week 3:** Deploy to production and monitor
6. **Week 4:** Team training and documentation handoff

**Estimated Effort:** 2-3 weeks with 1 engineer
**Risk Level:** Low (greenfield deployment, no existing systems affected)
**Deployment Type:** Greenfield (first deployment to production)
