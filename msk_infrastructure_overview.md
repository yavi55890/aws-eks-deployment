# MSK (Kafka) Infrastructure Overview

> **Document Purpose**: Comprehensive overview of the MSK infrastructure that powers event-driven automation from BigPanda incidents. Written to give a clear mental model of the architecture, security, networking, and operational processes -- even if you haven't directly built the infrastructure.
> **Source**: [Confluence: MSK (Kafka)](https://ouryahoo.atlassian.net/wiki/spaces/OCE/pages/951943376/MSK+Kafka)
> **Created**: February 2026
> **Jira Epic**: [SRETOOLS-2384](https://ouryahoo.atlassian.net/browse/SRETOOLS-2384)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Why MSK Exists -- The Business Case](#why-msk-exists----the-business-case)
3. [End-to-End Data Flow](#end-to-end-data-flow)
4. [Cluster Architecture](#cluster-architecture)
   - [Dev vs. Prod Clusters](#dev-vs-prod-clusters)
   - [Cluster Configuration](#cluster-configuration)
   - [Terraform Directory Structure](#terraform-directory-structure)
5. [Authentication & Authorization](#authentication--authorization)
   - [IAM Authentication (AWS-native consumers)](#iam-authentication-aws-native-consumers)
   - [SCRAM Authentication (On-prem consumers)](#scram-authentication-on-prem-consumers)
   - [Kafka ACLs](#kafka-acls)
6. [Secrets Management](#secrets-management)
   - [Secret Lifecycle](#secret-lifecycle)
   - [CKMS to AWS Secrets Manager Sync](#ckms-to-aws-secrets-manager-sync)
   - [Secret Naming & Encryption](#secret-naming--encryption)
7. [Networking & Access Control](#networking--access-control)
   - [How On-Prem Traffic Reaches MSK](#how-on-prem-traffic-reaches-msk)
   - [YCPI Routing via Edgeman](#ycpi-routing-via-edgeman)
   - [Dynamic IP Allowlisting (MSD/WAKL)](#dynamic-ip-allowlisting-msdwakl)
   - [Security Groups & Prefix Lists](#security-groups--prefix-lists)
8. [Tenant Onboarding](#tenant-onboarding)
   - [Onboarding Workflow (Step-by-Step)](#onboarding-workflow-step-by-step)
   - [Tenant Configuration Fields](#tenant-configuration-fields)
9. [Topic & ACL Management](#topic--acl-management)
   - [Topic Defaults](#topic-defaults)
   - [ACL Types](#acl-types)
   - [The MSK Admin EC2 Instance](#the-msk-admin-ec2-instance)
10. [Operational Runbook](#operational-runbook)
    - [Useful AWS CLI Commands](#useful-aws-cli-commands)
    - [Troubleshooting Common Issues](#troubleshooting-common-issues)
11. [Key Decisions & Trade-offs](#key-decisions--trade-offs)
12. [Glossary](#glossary)

---

## Executive Summary

We run two Amazon MSK (Managed Streaming for Apache Kafka) clusters -- one for dev, one for prod -- that serve as the backbone for **event-driven automation from BigPanda incidents**. When a BigPanda alert is tagged with `bpi_msk_topic`, the incident payload is written to a Kafka topic. Downstream consumers (like the `kafka_consumer` in this repo) then process those events and execute automated remediation.

**Key numbers to know:**

| Metric | Value |
|---|---|
| **Clusters** | 2 (dev + prod) |
| **Region** | `us-east-1` |
| **Kafka Version** | 3.9.x |
| **Broker Instance Type** | `kafka.m7g.large` |
| **Brokers per Cluster** | 3 |
| **Auth Methods** | IAM + SCRAM |
| **Topic Retention** | 7 days |
| **Topic Replication Factor** | 3 |

The infrastructure is managed entirely via **Terraform** in the [sre-tooling-infra-terraform](https://git.ouryahoo.com/SRE/sre-tooling-infra-terraform/tree/main/terraform/msk) repo and deployed through **Screwdriver CI/CD** pipelines.

---

## Why MSK Exists -- The Business Case

Before MSK, there was no standardized way for teams to **programmatically react to BigPanda incidents** in real-time. Teams either polled APIs or relied on manual triage.

MSK solves this by providing a **real-time event stream**:

1. Any BigPanda alert can be routed to a Kafka topic just by adding a tag (`bpi_msk_topic`).
2. Tenants consume from their own dedicated topic -- isolated, authenticated, and authorized.
3. Consumers can run anywhere: **on-prem** (via SCRAM over YCPI) or **in AWS** (via IAM over PrivateLink).

This enables use cases like:
- **Automated remediation**: Trigger Screwdriver jobs or webhooks when specific incidents occur.
- **Alert enrichment**: Cross-reference incidents with ServiceNow, Rootly, or other systems.
- **Escalation workflows**: Automatically escalate to Rootly or create OPS tickets based on incident patterns.
- **Outage detection**: Analyze alert volume and patterns in real-time.

---

## End-to-End Data Flow

Here's the full journey of an incident from BigPanda to automated action:

```
┌─────────────┐     ┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│  BigPanda   │────▶│ API Gateway  │────▶│ Lambda         │────▶│ MSK Cluster  │
│  Incident   │     │              │     │ (Kafka         │     │ (Kafka       │
│  (tagged    │     │              │     │  Producer)     │     │  Topic)      │
│   w/ topic) │     │              │     │                │     │              │
└─────────────┘     └──────────────┘     └────────────────┘     └──────┬───────┘
                                                                       │
                                                              ┌────────▼───────┐
                                                              │  Kafka         │
                                                              │  Consumer      │
                                                              │  (this repo)   │
                                                              │                │
                                                              │  Executes:     │
                                                              │  - SD jobs     │
                                                              │  - Webhooks    │
                                                              │  - Slack msgs  │
                                                              │  - Escalations │
                                                              └────────────────┘
```

**Step by step:**

1. An alert fires in BigPanda with the `bpi_msk_topic` tag set to a topic name (e.g., `sre-tooling`).
2. BigPanda's `bpi-msk` integration sends the full incident payload to **API Gateway**.
3. API Gateway invokes the **`bpi-msk` Lambda function** (Kafka producer), which writes the payload to the Kafka topic specified in the tag.
4. The **Kafka consumer** (running on-prem or in AWS) polls the topic, evaluates the incident against configured conditions, and runs the appropriate actions.

This is a clean **producer-consumer decoupling** -- the producer (Lambda) doesn't know or care who's consuming, and consumers can be added or removed without touching the producer.

---

## Cluster Architecture

### Dev vs. Prod Clusters

We maintain **completely separate clusters** per environment. They share the same Terraform module but differ in their configuration values:

| Aspect | Dev | Prod |
|---|---|---|
| **AWS Account** | `590183801200` | `851725424001` |
| **Cluster Name** | `bpi-msk-dev` | `bpi-msk-prod` |
| **Athenz Domain** | `sre.tooling.dev` | `sre.tooling.prod` |
| **API Gateway** | Dev gateway | Prod gateway |

**Both clusters share:**
- Kafka version `3.9.x`
- Instance type `kafka.m7g.large`
- 3 broker nodes (matching 3 subnets/AZs for high availability)
- IAM + SCRAM authentication enabled
- TLS encryption (client-to-broker and inter-broker)
- KMS encryption at rest
- CloudWatch Logs (90-day retention)
- Prometheus metrics (JMX + Node exporters)

### Cluster Configuration

| Setting | Value | Why |
|---|---|---|
| **Public Access** | `SERVICE_PROVIDED_EIPS` | Required for on-prem consumers accessing via YCPI |
| **Multi-VPC Connectivity** | Enabled | Allows cross-account AWS consumers to connect via port 9098 |
| **Encryption in Transit** | TLS everywhere | Client-broker and inter-broker communication |
| **Encryption at Rest** | KMS | All data encrypted on disk |
| **Monitoring** | CloudWatch + Prometheus | Logs + JMX/Node metrics |

### Terraform Directory Structure

All infrastructure lives in the [sre-tooling-infra-terraform](https://git.ouryahoo.com/SRE/sre-tooling-infra-terraform/tree/main/terraform/msk) repo:

```
terraform/msk/
├── common/                        # Shared Terraform modules
│   ├── main.tf                    # Main cluster config (brokers, auth, encryption)
│   ├── kafka_resources.tf         # Topics and ACLs (reference/documentation only!)
│   ├── route53.tf                 # DNS records
│   ├── locals.tf                  # Shared variables
│   └── versions.tf                # Provider versions
└── environments/
    ├── dev-us-east-1/             # Dev environment
    │   ├── locals.tf              # Dev-specific values (account ID, cluster name, etc.)
    │   └── common-main.tf         # Symlinks/references to common modules
    └── prod-us-east-1/            # Prod environment
        ├── locals.tf              # Prod-specific values
        └── common-main.tf         # Symlinks/references to common modules
```

**Important nuance**: `kafka_resources.tf` defines topics and ACLs in Terraform **for documentation purposes only**. The actual topic/ACL management is done via shell scripts on the MSK Admin EC2 instance (more on that later). This is because Kafka ACLs don't play nicely with Terraform's state management.

---

## Authentication & Authorization

MSK supports two authentication paths, and which one applies depends on **where the consumer is connecting from**:

```
                        ┌─────────────────────────────┐
                        │        MSK Cluster           │
                        │                              │
   On-Prem Consumer ───▶│  Port 9196: SCRAM Auth      │◀── YCPI routing
   (via YCPI)           │  (username/password + ACLs)  │    (Edgeman rules)
                        │                              │
   AWS Consumer ───────▶│  Port 9098: IAM Auth         │◀── PrivateLink /
   (cross-account)      │  (IAM role + cluster policy) │    Multi-VPC
                        │                              │
                        └─────────────────────────────┘
```

### IAM Authentication (AWS-native consumers)

**When**: Consumer is running inside AWS (same or different account) and connecting via Multi-VPC Connectivity (PrivateLink) on **port 9098**.

**How it works**:
- Access is governed by the **MSK Cluster Policy** -- a resource-based IAM policy attached to the cluster.
- The policy explicitly lists which IAM role ARNs can read which topics and consumer groups.
- These policies are auto-generated from the `bigpanda-master-config.yaml` tenant configurations.

**Example cluster policy statement** (simplified):
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:iam::851725424001:role/msk-alert-consumer"
    ]
  },
  "Action": [
    "kafka-cluster:ReadData",
    "kafka-cluster:DescribeTopic",
    "kafka-cluster:*Group"
  ],
  "Resource": [
    "arn:aws:kafka:us-east-1:...:topic/.../sre-tooling",
    "arn:aws:kafka:us-east-1:...:group/.../sre-tooling*"
  ]
}
```

**Key point**: With IAM, there are no passwords -- the consumer assumes an IAM role, and the cluster policy decides what that role can do.

### SCRAM Authentication (On-prem consumers)

**When**: Consumer is running on-prem and connecting through YCPI on **port 9196** (public access).

**How it works**:
- Consumer authenticates with a **username and password** (SCRAM-SHA-512).
- Credentials are stored in AWS Secrets Manager and associated with the MSK cluster.
- After authentication, **Kafka ACLs** determine what the user can access.

**Flow**:
1. Consumer connects to MSK broker via YCPI routing (more on this below).
2. MSK fetches the secret from Secrets Manager and validates the SCRAM handshake.
3. Once authenticated, Kafka ACLs enforce topic/group access for that username.

### Kafka ACLs

ACLs are the authorization layer for SCRAM-authenticated users. Three types:

| ACL Type | Principal | Access |
|---|---|---|
| **Admin** | `User:msk-admin` | Full access to everything |
| **Broker** | `User:CN=*.{broker-domain}` | Full access for inter-broker replication |
| **Tenant** | `User:{tenant-key}` | Read + Describe on their topic; consumer group access (prefixed) |

Tenant ACL example:
```
Principal:  User:sre_tooling
Topic:      sre-tooling       → Read, Describe
Group:      sre_tooling-*     → All (prefixed pattern)
Cluster:                      → Describe
```

The prefixed consumer group pattern (`{tenant}-*`) means tenants can create any consumer group that starts with their name.

---

## Secrets Management

### Secret Lifecycle

Secrets go through a multi-system journey, which can be confusing at first. Here's the clear picture:

```
┌───────────┐     ┌──────────────────┐     ┌────────────────────┐     ┌─────────────┐
│  CKMS     │────▶│ Screwdriver      │────▶│ AWS Secrets        │────▶│ MSK Cluster │
│ (source   │     │ Pipeline         │     │ Manager            │     │ (uses creds │
│  of truth │     │ (sync script)    │     │ (credentials       │     │  for SCRAM) │
│  for pwd) │     │                  │     │  stored here)      │     │             │
└───────────┘     └──────────────────┘     └────────────────────┘     └─────────────┘
      │
      │
      ▼
┌───────────────┐
│ Tenant        │
│ (retrieves    │
│  creds from   │
│  CKMS to use  │
│  for SCRAM)   │
└───────────────┘
```

**Why two secret stores?**
- **AWS Secrets Manager**: MSK *requires* credentials to be in Secrets Manager -- it's the only way to configure SCRAM auth. The cluster directly reads from Secrets Manager.
- **CKMS (Cloud Key Management Service)**: Yahoo's internal secret store, accessible from on-prem via Athenz certificates. Tenants retrieve their credentials from here.

Passwords are **generated once in CKMS** and **synced to Secrets Manager**. This way:
- The MSK cluster can read them (from Secrets Manager).
- The on-prem consumer can read them (from CKMS).
- There's a single source of truth for password generation.

### CKMS to AWS Secrets Manager Sync

The sync is handled by a **Screwdriver CI/CD pipeline** that:

1. Reads credentials from CKMS.
2. Updates the corresponding secret in AWS Secrets Manager.
3. Validates the secret format (`{"username": "...", "password": "..."}`).
4. Triggers secret association with the MSK cluster if needed.

This pipeline runs during tenant onboarding. Helper scripts live in `sre-terraform-infrastructure-bootstrap/helpers/`.

### Secret Naming & Encryption

| Item | Convention |
|---|---|
| **Admin Secret** | `AmazonMSK_bpi-msk-access-admin` |
| **Tenant Secret** | `AmazonMSK_bpi-msk-access-{tenant-key}` |
| **Tenant Key** | Tenant name, lowercased, spaces/dots → underscores |
| **KMS Key Alias** | `alias/msk-scram-secrets-{environment}` |

All secrets are encrypted with a dedicated KMS key whose policy allows both the MSK service and Secrets Manager to decrypt.

**Secret rotation**: Not currently automated. Manual process: generate new password in CKMS → update Secrets Manager → restart Kafka clients. (This is a known improvement area.)

---

## Networking & Access Control

This is probably the most complex part of the infrastructure. There are two very different network paths depending on where the consumer lives.

### How On-Prem Traffic Reaches MSK

On-prem consumers can't directly reach AWS resources. Traffic flows through YCPI (Yahoo Cloud Private Infrastructure):

```
┌──────────┐     ┌──────────┐     ┌───────────────┐     ┌─────────────┐
│ On-prem  │────▶│  YCPI    │────▶│ Public EIPs   │────▶│ MSK Broker  │
│ Consumer │     │ (routing │     │ (port 9196)   │     │ (SCRAM)     │
│          │     │  layer)  │     │               │     │             │
└──────────┘     └──────────┘     └───────────────┘     └─────────────┘
```

### YCPI Routing via Edgeman

[Edgeman](https://edgeman.ouryahoo.com) is the self-service UI for managing YCPI routing rules. Our rules live under the **SRE_US** property.

YCPI created **6 DNS records** in the `yahooinc.com` domain -- one per broker, per environment:

```
broker[1,2,3]-public-<cluster-name>.yahooinc.com
```

These DNS records point to the YCPI edge. Edgeman routes are configured as **"partial blind tunnels"** (note the `P:` prefix on tunnel routes), which forward the TCP connection through to the actual MSK broker public EIPs.

**In plain English**: When an on-prem consumer connects to `broker1-public-bpi-msk-prod.yahooinc.com:9196`, YCPI forwards that connection to the real MSK broker's public IP. The `P:` prefix means it's a "partial" blind tunnel -- YCPI routes the traffic without inspecting it, but does enforce network-level access controls.

Edgeman rules reference: [https://edgeman.ouryahoo.com/edgeman/showRevs/269/](https://edgeman.ouryahoo.com/edgeman/showRevs/269/)

### Dynamic IP Allowlisting (MSD/WAKL)

Even though the MSK brokers have public EIPs, we don't want the whole internet hitting them. Only **YCPI IP ranges** should be allowed.

The problem: YCPI IP ranges change over time. The solution: **WAKL (Wide Area Key List)** -- a Lambda function that runs every hour and updates AWS security groups with the latest YCPI IPs.

```
┌──────────────┐     ┌────────────────────┐     ┌──────────────────┐     ┌───────────────┐
│ MSD          │────▶│ WAKL Lambda        │────▶│ EC2 Managed      │────▶│ Security      │
│ (source of   │     │ (runs hourly)      │     │ Prefix Lists     │     │ Groups on     │
│  YCPI IPs)   │     │                    │     │ (2 lists,        │     │ MSK Brokers   │
│              │     │                    │     │  250 IPs each)   │     │ (port 9196)   │
└──────────────┘     └────────────────────┘     └──────────────────┘     └───────────────┘
```

**How it works:**

1. **MSD (Micro Segmentation Daemon)** defines transport policies that declare: "YCPI networks are allowed to reach our MSK service on port 9196 via TCP."
2. **WAKL Lambda** runs every hour, queries MSD for the current YCPI IP ranges, and writes them into **EC2 Managed Prefix Lists**.
3. **Security Groups** attached to the MSK brokers reference these prefix lists. When the prefix lists update, the security groups automatically reflect the new IPs.

### Security Groups & Prefix Lists

| Component | Count | Details |
|---|---|---|
| **Prefix Lists** | 2 | Max 250 entries each (handles large IP ranges) |
| **Security Groups** | 2 | One per prefix list, ingress on port 9196/TCP |
| **WAKL Lambda Frequency** | Every hour | `rate(1 hour)` CloudWatch Events schedule |

The MSD transport policy in Terraform looks like:
```
Direction:       ingress
Remote Service:  ycpi.ycpi-networks-access
Destination Port: 9196
Source Ports:     1024-65535
Protocol:        TCP
Enforcement:     enforce
Scope:           aws
```

**The Athenz service for MSD**: `bpi-msk-access` under domain `sre.tooling.{environment}`. You can view MSD policies in the Athenz UI under the Microsegmentation tab.

---

## Tenant Onboarding

### Onboarding Workflow (Step-by-Step)

Here's the full onboarding lifecycle when a new team wants to consume from MSK:

```
 ① Update tenant config          ② Upload config to S3
    in bigpanda-master-               (Screwdriver pipeline)
    config.yaml                   
         │                                  │
         ▼                                  ▼
 ③ Terraform apply               ④ Generate SCRAM credentials
    (creates AWS resources:           in CKMS
     Secrets Manager secret,     
     IAM roles, Athenz roles)         │
         │                            ▼
         │                     ⑤ Sync creds to AWS
         │                        Secrets Manager
         │                        (Screwdriver pipeline)
         │                            │
         ▼                            ▼
 ⑥ Associate secrets with     ⑦ Create Kafka topics & ACLs
    MSK cluster                   on MSK Admin EC2 instance
    (AWS CLI command)             (msk-setup.sh script)
```

**Detailed steps:**

1. **Update tenant config** in [bigpanda-master-config.yaml](https://git.ouryahoo.com/SRE/bigpanda-infra/blob/main/configs/bigpanda-master-config.yaml):
   ```yaml
   - name: YourTenantName
     athenz:
       domain: your.domain
       group: your.group
     msk:
       topic_needed: true
       trusted_arns:              # Optional: for IAM/PrivateLink access
         - arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME
   ```

2. **Upload config to S3** via the [bigpanda-infra Screwdriver pipeline](https://screwdriver.ouroath.com/pipelines/1072346/).
   - Bucket: `{account-id}-sre-tooling-{env}-bigpanda-configs-{region}`
   - Key: `configs/bigpanda-master-config.yaml`

3. **Run Terraform** via the [sre-tooling-infra-terraform Screwdriver pipeline](https://screwdriver.ouroath.com/pipelines/1067527/). Terraform reads the S3 config and auto-creates:
   - Secrets Manager secret container (empty initially)
   - IAM consumer role
   - Athenz roles (if `athenz.domain` and `athenz.group` are specified)

4. **Generate SCRAM credentials** in CKMS (handled by bigpanda-infra onboarding process).

5. **Sync credentials** from CKMS to AWS Secrets Manager (Screwdriver pipeline).

6. **Associate secrets with the cluster**:
   ```bash
   aws kafka batch-associate-scram-secret \
     --cluster-arn <cluster-arn> \
     --secret-arn-list <secret-arns>
   ```
   (The exact command is provided in Terraform output as `msk_scram_association_command`.)

7. **Create topics and ACLs** on the MSK Admin EC2 instance using `msk-setup.sh`.

### Tenant Configuration Fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Tenant identifier |
| `athenz.domain` | No | Athenz domain for OIDC access |
| `athenz.group` | No | Athenz group for role membership |
| `msk.topic_needed` | Yes | Set to `true` to provision a Kafka topic |
| `msk.trusted_arns` | No | IAM role ARNs for direct cross-account access |

---

## Topic & ACL Management

### Topic Defaults

Every tenant topic is created with the same settings:

| Setting | Value | Rationale |
|---|---|---|
| **Partitions** | 3 | Matches broker count for even distribution |
| **Replication Factor** | 3 | Full replication across all brokers for durability |
| **Min In-Sync Replicas** | 2 | Allows one broker to be down without blocking writes |
| **Retention** | 7 days (`604800000` ms) | Enough time for consumers to catch up after outages |
| **Cleanup Policy** | `delete` | Old messages are deleted after retention period |

### ACL Types

Three categories of ACLs are configured:

**1. Admin ACLs** -- `User:msk-admin`
- Full wildcard access to all topics, groups, and cluster operations.
- Used for management tasks from the EC2 admin instance.

**2. Broker ACLs** -- `User:CN=*.{broker-domain}`
- Full access for inter-broker communication.
- Required for partition replication and leader election.

**3. Tenant ACLs** -- `User:{tenant-key}`
- Read + Describe on their specific topic.
- All operations on consumer groups matching the prefix `{tenant-key}-*`.
- Describe access on the cluster (needed for metadata operations).

### The MSK Admin EC2 Instance

Since Kafka CLI tools (`kafka-topics.sh`, `kafka-acls.sh`) need direct network access to brokers, we run a dedicated **EC2 instance in the same VPC** as the MSK cluster.

- **Terraform**: [sre-tooling-infra-terraform/terraform/msk-ec2](https://git.ouryahoo.com/SRE/sre-tooling-infra-terraform/tree/main/terraform/msk-ec2)
- **Setup script**: [msk-setup.sh](https://git.ouryahoo.com/SRE/sre-tooling-infra-terraform/blob/main/helpers/msk-setup.sh) -- executed by Screwdriver on the EC2 instance
- **Auth**: IAM authentication to the cluster
- **Purpose**: Create topics, configure ACLs for SCRAM users, configure broker ACLs

This EC2 instance is **not** a consumer -- it's purely an admin tool for Kafka resource management.

---

## Operational Runbook

### Useful AWS CLI Commands

**Get bootstrap brokers** (connection endpoints):
```bash
aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>
```

**List SCRAM secrets associated with the cluster**:
```bash
aws kafka list-scram-secrets --cluster-arn <cluster-arn>
```

**Describe cluster** (full configuration):
```bash
aws kafka describe-cluster-v2 --cluster-arn <cluster-arn>
```

**Get cluster policy** (IAM access rules):
```bash
aws kafka get-cluster-policy --cluster-arn <cluster-arn> | jq -r .Policy | jq .
```

**Associate new SCRAM secrets**:
```bash
aws kafka batch-associate-scram-secret \
  --cluster-arn <cluster-arn> \
  --secret-arn-list <secret-arns>
```

### Troubleshooting Common Issues

**"Consumer can't connect from on-prem"**
1. Check WAKL Lambda execution logs in CloudWatch -- did it run successfully?
2. Check prefix list entries -- do they contain current YCPI IPs?
3. Check security group rules -- do they reference the prefix lists?
4. Check MSD transport policy status in Athenz UI.
5. Verify Edgeman routing rules for the broker DNS records.

**"Consumer authenticated but can't read topic"**
- For SCRAM: Check Kafka ACLs on the admin EC2 instance. The tenant's ACL might be missing or misconfigured.
- For IAM: Check the MSK cluster policy. The consumer's IAM role ARN must be listed with the correct topic resource.

**"Secret association failed"**
- Verify the secret format is exactly `{"username": "...", "password": "..."}` (no extra fields).
- Verify the KMS key policy allows the MSK service to decrypt.
- Check that the secret name starts with `AmazonMSK_` (MSK requirement).

**"New cluster deployment is failing"**
- MSK cluster creation is finicky. Follow the exact order:
  1. Deploy with public access **disabled** and multi-VPC **disabled**.
  2. Enable public access (`SERVICE_PROVIDED_EIPS`) in a separate `terraform apply`.
  3. Enable multi-VPC connectivity in another separate `terraform apply`.
  4. Symlink `route53.tf` and apply DNS last.
- Don't try to enable everything in one shot -- MSK doesn't like it.

---

## Key Decisions & Trade-offs

These are things leadership might ask about and the reasoning behind the choices:

**Q: Why Amazon MSK instead of self-managed Kafka or another streaming service?**
> MSK is fully managed -- no broker patching, no ZooKeeper ops, built-in IAM integration. We get the Kafka API (which teams already know) without the operational burden. The alternative was Kinesis, but Kafka's consumer group model and ecosystem tooling made it a better fit.

**Q: Why both IAM and SCRAM? Why not just one?**
> IAM only works for AWS-based consumers connecting via PrivateLink. On-prem consumers can't assume IAM roles, so they need username/password auth (SCRAM). We need both because we have consumers in both environments.

**Q: Why are ACLs managed via scripts instead of Terraform?**
> Kafka ACLs don't have great Terraform provider support, and managing them through Terraform's state can lead to drift issues since ACLs are a Kafka-side resource, not an AWS resource. The shell script approach is more reliable and easier to debug.

**Q: Why two prefix lists instead of one?**
> Each EC2 Managed Prefix List has a max of 250 entries. YCPI has more IP ranges than that, so we split across two prefix lists (and two corresponding security groups).

**Q: Why 7-day retention?**
> It's a balance between giving consumers enough buffer to recover from outages (weekend + a couple of days) without running up storage costs. Most consumers process messages within minutes, so 7 days is very generous.

**Q: What about secret rotation?**
> Currently manual. Automated rotation is a known TODO. The process would be: generate new password in CKMS → sync to Secrets Manager → restart consumers. The impact is low since credential compromise risk is mitigated by network-level controls (YCPI-only access).

**Q: What's the disaster recovery story?**
> The clusters are single-region (`us-east-1`) with 3 brokers across 3 AZs. We tolerate single-AZ failures. Full regional failure is accepted risk for this use case -- incidents would still be in BigPanda, and automation would resume once the cluster is back. Cross-region replication is not currently configured.

---

## Glossary

| Term | Definition |
|---|---|
| **MSK** | Amazon Managed Streaming for Apache Kafka -- AWS's managed Kafka service |
| **SCRAM** | Salted Challenge Response Authentication Mechanism -- username/password auth for Kafka |
| **IAM** | AWS Identity and Access Management -- role-based auth for AWS services |
| **YCPI** | Yahoo Cloud Private Infrastructure -- the network layer connecting on-prem to cloud |
| **Edgeman** | Self-service UI for managing YCPI routing rules |
| **MSD** | Micro Segmentation Daemon -- defines network access policies |
| **WAKL** | Wide Area Key List -- Lambda that syncs YCPI IPs to AWS prefix lists |
| **CKMS** | Cloud Key Management Service -- Yahoo's internal secret store |
| **Athenz** | Yahoo's open-source identity and access management platform |
| **ACL** | Access Control List -- Kafka's native authorization mechanism |
| **Prefix List** | AWS EC2 Managed Prefix List -- a named set of IP ranges used in security groups |
| **PrivateLink** | AWS service for private connectivity between VPCs without traversing the internet |
| **Multi-VPC Connectivity** | MSK feature that allows cross-account access via PrivateLink (port 9098) |
| **EIP** | Elastic IP -- static public IP address assigned to MSK brokers |
| **Screwdriver** | Yahoo's CI/CD platform (similar to Jenkins/GitHub Actions) |
| **BigPanda** | AIOps platform for incident management and alert correlation |
| **Rootly** | Incident management platform for escalations |
