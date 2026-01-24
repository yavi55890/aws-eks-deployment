# A Beginner's Guide to Kubernetes for AWS EKS Deployment

## Introduction: What is Kubernetes?

Think of Kubernetes as an **intelligent orchestrator for containerized applications**—like a conductor managing a complex orchestra. Just as a conductor ensures musicians have the right instruments, sheet music, and synchronization, Kubernetes ensures your containerized applications have the right resources, are discoverable, and stay running reliably.

Kubernetes (often abbreviated as K8s, where 8 represents the eight letters between K and S) is an **open-source platform for managing containerized workloads and services**. The name comes from Greek, meaning "helmsman" or "pilot"—reflecting its role as the captain steering your distributed systems.

### Why You Need Kubernetes

In a simple world, you might run a few containers on a single machine. But in production, you face real challenges:

- **Containers fail** and need to be automatically restarted
- **Traffic spikes** require scaling applications up quickly
- **Updates** need to happen without downtime
- **Resources** need to be allocated efficiently across multiple machines
- **Networking** becomes complex with services discovering each other dynamically

Kubernetes solves these challenges through **automation and declarative configuration**. Instead of manually managing containers, you declare what you want ("I want 3 copies of my web app running"), and Kubernetes makes it happen and keeps it that way.

---

## Core Kubernetes Concepts

### 1. Workloads: How Your Applications Run

Kubernetes provides several ways to run applications, each suited for different purposes:

#### Pods (The Building Block)

A **Pod** is the smallest unit in Kubernetes—like a shipping container on a cargo ship. One Pod typically contains a single application container, but it can contain multiple tightly-coupled containers that share networking.

**Key characteristics:**
- Containers in a Pod share a network interface, so they can communicate via `localhost`
- Short-lived by nature (containers are ephemeral)
- **You don't usually create Pods directly**—instead, you use higher-level controllers that manage Pods for you

**Analogy:** If a Docker container is like a single appliance (a refrigerator), a Pod is like a small kitchen where appliances (containers) work together.

#### Deployments (For Stateless Apps)

A **Deployment** is the go-to choice for running stateless applications. It manages a set of identical, interchangeable Pods with these capabilities:

- **Scaling:** Run 3 copies, or 100 copies—change it anytime
- **Rolling updates:** Deploy new versions without downtime
- **Self-healing:** If a Pod crashes, Deployment automatically creates a replacement
- **History tracking:** Roll back to previous versions if needed

**Best for:** Web servers, API services, microservices, anything that doesn't need to persist data in a specific location.

**Example use case:** Your Kafka consumer application runs as a Deployment with 1 replica (due to file-based state limitations).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3              # Run 3 copies
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: my-app:v1.0
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

#### StatefulSets (For Stateful Apps)

A **StatefulSet** is designed for applications that need to maintain state and persistent identity:

- Each Pod has a stable, predictable hostname (e.g., `mysql-0`, `mysql-1`)
- Each Pod is paired with its own persistent storage
- Pods are created/destroyed in order, making coordination easier
- Good for replicating data across multiple instances

**Best for:** Databases (PostgreSQL, MySQL), distributed systems (Cassandra, ZooKeeper), message queues that need persistence.

**Why not just use Deployment?** Imagine a database with 3 replicas. When a Pod restarts, you need to know which replica came back online and which data it owns. StatefulSets provide this ordered, stateful management.

#### Jobs & CronJobs (For One-off or Scheduled Tasks)

- **Job:** Runs a task to completion, then stops (useful for batch processing, backups)
- **CronJob:** Runs Jobs on a schedule (like Linux cron)

**Best for:** Data processing, cleanup tasks, scheduled reports, backups.

#### DaemonSets (Run on Every Node)

A **DaemonSet** ensures a Pod runs on every node in your cluster (or matching nodes). Useful for infrastructure concerns.

**Best for:** Log collectors, monitoring agents, networking plugins.

---

### 2. Services: Exposing Your Applications

Here's a key insight: **Pods are temporary**. They're created, destroyed, and replaced constantly. How do other applications find them?

A **Service** is like a **load balancer + stable name directory**. It provides:

- A stable, long-lived IP address or DNS name
- Load balancing across multiple Pod replicas
- Automatic discovery without needing to know Pod IPs

#### Service Types

**ClusterIP (Default)**
- Accessible only within the cluster
- Apps inside the cluster can find each other via stable names
- Example: Your web app finds your database via `postgres:5432`
- No external access

**NodePort**
- Accessible from outside the cluster on a specific port on each Node
- Good for development/testing
- Port range: 30000-32767
- Example: `http://your-node-ip:30123`

**LoadBalancer**
- Integrates with cloud provider's load balancer (AWS ELB/ALB)
- Gets a public IP address automatically
- Best for exposing your primary application to the internet
- Each Service creates a separate load balancer (can be expensive at scale)

**ExternalName**
- Maps a Service to an external DNS name
- Useful for integrating with external systems outside the cluster

#### Ingress: HTTP/HTTPS Routing

Instead of creating separate load balancers for each service, **Ingress** provides HTTP/HTTPS routing:

- Single entry point for multiple services
- Route by hostname and path (e.g., `api.example.com` vs `web.example.com`)
- Automatic TLS/SSL certificates
- More efficient than LoadBalancer Services

**Example:** Route `example.com/api` to your API service and `example.com/web` to your web service using a single load balancer.

#### Networking Inside the Cluster

Kubernetes provides automatic networking:
- Every Pod gets a unique IP address
- Pods can communicate directly across nodes (no NAT)
- No need to manually configure port mappings
- The cluster networking layer handles all the complexity

**NetworkPolicy** allows you to restrict traffic:
- Control which Pods can talk to each other
- Control traffic to/from the outside world
- Acts like a firewall at the Pod level

---

### 3. Storage: Persisting Data

Pods are temporary, but often your data isn't. Kubernetes provides a hierarchy of storage concepts:

#### Volumes: Basic Storage

A **Volume** is temporary storage mounted into a Pod. Types include:

- **emptyDir:** Temporary storage tied to Pod's lifetime (deleted when Pod ends)
- **hostPath:** Storage on the node's filesystem
- **configMap/secret:** Config/secret data mounted as files
- **cloud storage:** AWS EBS, EFS, S3, etc.

#### PersistentVolumes (PV) & PersistentVolumeClaims (PVC)

This is where real persistence happens. Think of it like a rental system:

**Administrators create PersistentVolumes:**
- "We have 100GB of AWS EBS storage available"
- Define storage type, size, and access rules

**Users create PersistentVolumeClaims:**
- "I need 20GB of storage"
- Kubernetes matches the claim to an available volume
- Pod uses the PVC to access actual storage

**StorageClasses enable dynamic provisioning:**
- Instead of pre-creating PVs, define storage types (fast SSD, slow HDD)
- When you request a PVC with a StorageClass, Kubernetes automatically provisions the storage
- Common in cloud environments (AWS uses EBS)

**Example workflow:**
```yaml
# Admin creates a StorageClass
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3  # AWS EBS fast SSD

---

# User creates a PersistentVolumeClaim
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi

---

# Pod uses the PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  template:
    spec:
      containers:
      - name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data
```

**Key insight for AWS EKS:** AWS automatically provides EBS volumes as storage. When you create a PVC, Kubernetes can automatically provision an EBS volume for you.

---

### 4. Configuration: Managing Settings & Secrets

Your application needs configuration (database URLs, feature flags) and secrets (passwords, API keys). Kubernetes provides two resources:

#### ConfigMaps: Non-Sensitive Configuration

Store application settings without rebuilding your container image:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_TIMEOUT: "30s"
  LOG_LEVEL: "INFO"
  config.ini: |
    [server]
    port=8080
    debug=false
```

**Usage:**
- Pass as environment variables
- Mount as files in containers
- Reference from command-line arguments

**Advantage:** Change configuration without rebuilding or redeploying your Docker image.

#### Secrets: Sensitive Data

Store passwords, API keys, and sensitive data:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: dXNlcm5hbWU=  # base64-encoded "username"
  password: cGFzc3dvcmQ=  # base64-encoded "password"
```

**Security best practices:**
- Enable encryption at rest in your cluster
- Limit who can view/modify Secrets (RBAC)
- Don't commit Secrets to git—use external secret management
- Consider external tools like AWS Secrets Manager or HashiCorp Vault

**Usage:** Same as ConfigMaps—environment variables or mounted files.

---

### 5. Security: Controlling Access & Protecting Workloads

Kubernetes security operates at multiple layers:

#### ServiceAccounts: Identity for Pods

Pods need identity to access Kubernetes APIs or external services:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: app-sa  # This pod runs with app-sa identity
  containers:
  - name: app
    image: my-app
```

**Use case:** Your Kafka consumer needs to access AWS services—it uses a ServiceAccount that grants those permissions.

#### RBAC: Role-Based Access Control

Control who can do what in your cluster:

- **Role:** Define permissions (can create Pods, read Secrets, etc.)
- **RoleBinding:** Assign a Role to a user or ServiceAccount
- **ClusterRole:** Like Role but cluster-wide
- **ClusterRoleBinding:** Cluster-wide role assignment

**Example: Give a ServiceAccount permission to read ConfigMaps**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-config
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: config-reader
subjects:
- kind: ServiceAccount
  name: app-sa
```

#### Pod Security Standards

Control what Pods can do (like sandboxing):

- **Restricted:** Most secure, prevents privileged operations
- **Baseline:** Minimal restrictions, allows common scenarios
- **Privileged:** No restrictions

Restricts things like:
- Running as root
- Using host networking
- Mounting host filesystems
- Capabilities escalation

#### Network Policies: Firewall Rules

Control traffic between Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-only
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 8080
```

This says: "The backend app only accepts traffic from the API app on port 8080."

---

### 6. Resource Management: Scheduling & Limits

Kubernetes needs to know how much resources your application needs and should be allowed to use:

#### Resource Requests (Minimum Requirements)

Tell Kubernetes what your container needs:

```yaml
containers:
- name: app
  resources:
    requests:
      cpu: "250m"        # 250 millicores (0.25 CPU)
      memory: "128Mi"    # 128 megabytes
```

**How it's used:**
- Scheduler uses requests to decide which node can run your Pod
- Kubernetes guarantees at least this much resource
- Helps prevent oversubscription

**Measurement:**
- **CPU:** 1 full CPU = "1000m" (millicores)
- **Memory:** Mi (mebibytes), Gi (gibibytes), etc.

#### Resource Limits (Maximum Allowed)

Prevent containers from hogging resources:

```yaml
containers:
- name: app
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "500m"        # Can't use more than 0.5 CPU
      memory: "256Mi"    # Can't use more than 256Mi RAM
```

**What happens if limits are exceeded:**
- **CPU:** Container is throttled (slowed down)
- **Memory:** Container may be killed and restarted

#### Quality of Service (QoS) Classes

Kubernetes automatically assigns QoS classes based on requests/limits:

1. **Guaranteed** (requests = limits)
   - Highest priority
   - Last to be evicted if node runs out of resources
   - Best for critical services

2. **Burstable** (has requests AND limits, but not equal)
   - Medium priority
   - Can burst above requests if resources available

3. **BestEffort** (no requests/limits)
   - Lowest priority
   - First to be evicted
   - For non-critical workloads

**Best practice for your Kafka consumer:** Set both requests and limits equal to get "Guaranteed" QoS so it isn't evicted unexpectedly.

---

## How It All Works Together: A Complete Example

Let's deploy your Kafka consumer application to EKS:

```yaml
---
# 1. Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-consumer-config
data:
  log-level: "INFO"
  kafka-topic: "incidents"

---
# 2. Secrets for credentials
apiVersion: v1
kind: Secret
metadata:
  name: kafka-credentials
type: Opaque
stringData:
  username: kafka-user
  password: secret-password-here

---
# 3. ServiceAccount for the application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka-consumer-sa

---
# 4. Role to read ConfigMaps and Secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]

---
# 5. Bind the role to the ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-config
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: config-reader
subjects:
- kind: ServiceAccount
  name: kafka-consumer-sa

---
# 6. The Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
spec:
  replicas: 2                    # Run 2 copies for redundancy
  selector:
    matchLabels:
      app: kafka-consumer
  template:
    metadata:
      labels:
        app: kafka-consumer
    spec:
      serviceAccountName: kafka-consumer-sa
      containers:
      - name: consumer
        image: my-company/kafka-consumer:v1.0
        ports:
        - containerPort: 8080

        # Configuration from ConfigMap
        envFrom:
        - configMapRef:
            name: kafka-consumer-config

        # Secrets injected as environment variables
        env:
        - name: KAFKA_USERNAME
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: username
        - name: KAFKA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: password

        # Resource management
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"

        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

---
# 7. Expose via Service (internal to cluster)
apiVersion: v1
kind: Service
metadata:
  name: kafka-consumer
spec:
  type: ClusterIP
  selector:
    app: kafka-consumer
  ports:
  - port: 8080
    targetPort: 8080
```

**What this does:**

1. **ConfigMap** stores non-secret config
2. **Secret** stores credentials (encrypted)
3. **ServiceAccount** gives the Pod identity
4. **RBAC** allows the Pod to read its config
5. **Deployment** ensures 2 copies of the consumer always run
6. **Resource requests** help Kubernetes schedule optimally
7. **Health probes** detect failures and restart
8. **Service** provides internal networking

---

## AWS EKS-Specific Concepts

### IAM Integration (IRSA: IAM Roles for Service Accounts)

In AWS EKS, you can grant Pods permissions to AWS services without embedding credentials:

1. Create an IAM Role with your required permissions
2. Create a ServiceAccount in Kubernetes
3. Establish a trust relationship between them
4. Pods with this ServiceAccount can assume the IAM role

**Example: Kafka consumer accessing CKMS**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka-consumer-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/kafka-consumer-role
```

When a Pod uses this ServiceAccount, it automatically gets AWS credentials to assume the IAM role.

### Storage: EBS Integration

EKS integrates with AWS EBS for persistent storage:

```yaml
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
```

When you create a PVC with this StorageClass, Kubernetes automatically provisions an EBS volume.

### Load Balancer Integration

EKS automatically integrates with AWS Load Balancers:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

Kubernetes automatically provisions an AWS ELB and configures it to route traffic to your Pods.

---

## Key Concepts Summary Table

| Concept | Purpose | Analogy |
|---------|---------|---------|
| **Pod** | Smallest unit, container wrapper | Shipping container |
| **Deployment** | Manage Pods with scaling/updates | Fleet manager |
| **StatefulSet** | Manage stateful Pods with identity | Database cluster |
| **Job** | One-off tasks | Task scheduler |
| **Service** | Stable network endpoint | Load balancer + DNS |
| **Ingress** | HTTP routing | Reverse proxy |
| **Volume** | Temporary storage | Scratch disk |
| **PVC** | Persistent storage request | Storage allocation request |
| **ConfigMap** | Configuration storage | Config files |
| **Secret** | Sensitive data storage | Encrypted vault |
| **ServiceAccount** | Pod identity | User account |
| **Role/RoleBinding** | Access control | Permission assignment |
| **NetworkPolicy** | Traffic control | Firewall rules |
| **Resource Requests** | Minimum resources needed | Minimum requirements |
| **Resource Limits** | Maximum resources allowed | Resource caps |

---

## Practical Tips for Your Kafka Consumer Deployment

### 1. Use Namespaces for Organization

Create a namespace for your application:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kafka-consumer
```

Then deploy everything in that namespace:
```bash
kubectl apply -f deployment.yaml -n kafka-consumer
```

### 2. Use Labels Consistently

Labels help you organize and query resources:
```yaml
metadata:
  labels:
    app: kafka-consumer
    component: consumer
    env: production
    team: sre
```

Query with labels:
```bash
kubectl get pods -l app=kafka-consumer,env=production
```

### 3. Health Checks Are Critical

Always define liveness and readiness probes:
- **Liveness:** "Is my app alive?" (restart if fails)
- **Readiness:** "Is my app ready to serve traffic?" (remove from service if fails)

For your Kafka consumer:
```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - ps aux | grep kafka_consumer
  initialDelaySeconds: 60
  periodSeconds: 30
```

### 4. Set Resource Requests and Limits

Always specify both for production workloads:
```yaml
resources:
  requests:
    cpu: "500m"      # I need at least 0.5 CPU
    memory: "1Gi"    # I need at least 1GB RAM
  limits:
    cpu: "1000m"     # Don't let me use more than 1 CPU
    memory: "2Gi"    # Don't let me use more than 2GB RAM
```

### 5. Use ConfigMaps for Environment-Specific Config

Don't hardcode config in your Docker image:
```yaml
# development-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-consumer-config
data:
  environment: "development"
  kafka-brokers: "kafka-dev:9092"
  log-level: "DEBUG"

---
# production-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-consumer-config
data:
  environment: "production"
  kafka-brokers: "kafka-prod:9092"
  log-level: "INFO"
```

### 6. Monitor Your Application

Key metrics to watch:
- Pod restarts (indicates crashes)
- CPU/memory usage
- Request/limit ratios
- Health check failures
- Application-specific metrics (Kafka lag, message processing rate)

Use tools like:
- AWS CloudWatch Container Insights
- Prometheus + Grafana
- EKS built-in monitoring

---

## Common Kubernetes Commands

```bash
# View pods
kubectl get pods
kubectl get pods -n kafka-consumer
kubectl get pods -l app=kafka-consumer

# View pod details
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -f  # follow logs
kubectl logs <pod-name> --previous  # logs from crashed container

# Execute commands in a pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec <pod-name> -- ps aux

# View deployments
kubectl get deployments
kubectl describe deployment kafka-consumer

# Scale a deployment
kubectl scale deployment kafka-consumer --replicas=3

# Update a deployment
kubectl set image deployment/kafka-consumer consumer=my-app:v2.0
kubectl rollout status deployment/kafka-consumer
kubectl rollout undo deployment/kafka-consumer  # rollback

# View services
kubectl get services
kubectl describe service kafka-consumer

# View config and secrets
kubectl get configmaps
kubectl get secrets
kubectl describe configmap kafka-consumer-config

# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f .  # apply all YAML files in directory

# Delete resources
kubectl delete pod <pod-name>
kubectl delete deployment kafka-consumer
kubectl delete -f deployment.yaml
```

---

## Troubleshooting Common Issues

### Pod is Pending

**Symptom:** Pod stuck in `Pending` state

**Possible causes:**
- Not enough CPU/memory on any node
- PVC can't be bound (storage issue)
- Image pull error
- Node selector/affinity not matching

**Debug:**
```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Pod is CrashLoopBackOff

**Symptom:** Pod keeps restarting

**Possible causes:**
- Application crashes on startup
- Liveness probe failing
- Missing configuration or secrets

**Debug:**
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```

### Can't Access Application

**Symptom:** Service not reachable

**Possible causes:**
- Service selector doesn't match pod labels
- Readiness probe failing
- NetworkPolicy blocking traffic
- Firewall/security group issue

**Debug:**
```bash
kubectl get svc
kubectl describe svc <service-name>
kubectl get endpoints <service-name>
kubectl logs <pod-name>
```

### Out of Memory (OOMKilled)

**Symptom:** Pod killed due to memory

**Possible causes:**
- Memory limit too low
- Application has memory leak
- Unexpected load

**Debug:**
```bash
kubectl describe pod <pod-name>  # Look for "OOMKilled"
kubectl top pod <pod-name>        # Check current memory usage
```

**Fix:** Increase memory limits or investigate memory leak.

---

## Next Steps for Deploying to EKS

1. **Create an EKS cluster** on AWS (managed Kubernetes)
2. **Configure kubectl** to access your cluster
3. **Create namespaces** to organize your deployments
4. **Deploy your application** using YAML manifests
5. **Set up monitoring/logging** (CloudWatch integration)
6. **Configure autoscaling** (Horizontal Pod Autoscaler, Cluster Autoscaler)
7. **Set up CI/CD** to automate deployments
8. **Plan for backup/disaster recovery**

---

## Additional Resources

- **Official Kubernetes Docs:** https://kubernetes.io/docs/concepts/
- **AWS EKS Documentation:** https://docs.aws.amazon.com/eks/
- **Kubernetes Best Practices:** https://kubernetes.io/docs/concepts/configuration/overview/
- **Security Best Practices:** https://kubernetes.io/docs/concepts/security/
- **kubectl Cheat Sheet:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/

---

This guide provides the foundational concepts you need to understand and deploy applications on Kubernetes. As you gain experience, you'll explore more advanced topics like networking policies, custom resources, and operators. For now, focus on mastering these core concepts—they'll serve you well for 90% of application deployments!
