# Disaster Recovery (DR) on AWS
## Complete Reference for DevOps & DevSecOps Engineers

> **Scope:** What DR is, how it differs from backup, the four AWS DR strategies, every major AWS DR service in depth, database DR, Kubernetes DR, network-level failover, DR automation with Terraform and CI/CD, security during failover, testing with chaos engineering, and interview preparation.

---

## Table of Contents

1. [What Is Disaster Recovery?](#1-what-is-disaster-recovery)
2. [Core DR Metrics: RTO, RPO, and Beyond](#2-core-dr-metrics-rto-rpo-and-beyond)
3. [DR vs. Backup vs. High Availability](#3-dr-vs-backup-vs-high-availability)
4. [The Four AWS DR Strategies](#4-the-four-aws-dr-strategies)
5. [AWS Elastic Disaster Recovery (EDR/DRS)](#5-aws-elastic-disaster-recovery-edrdrs)
6. [Database Disaster Recovery](#6-database-disaster-recovery)
7. [Kubernetes & Container DR (EKS + Velero)](#7-kubernetes--container-dr-eks--velero)
8. [Storage DR (S3, EFS, EBS)](#8-storage-dr-s3-efs-ebs)
9. [Network-Level DR (Route 53, Global Accelerator)](#9-network-level-dr-route-53-global-accelerator)
10. [AWS Backup — Centralised Backup Management](#10-aws-backup--centralised-backup-management)
11. [DR in the Context of Cloud Migration](#11-dr-in-the-context-of-cloud-migration)
12. [DevOps Engineering Responsibilities for DR](#12-devops-engineering-responsibilities-for-dr)
13. [DevSecOps: Security During Failover](#13-devsecops-security-during-failover)
14. [Terraform for DR Infrastructure](#14-terraform-for-dr-infrastructure)
15. [DR Runbooks and Automation](#15-dr-runbooks-and-automation)
16. [Testing DR: Chaos Engineering & GameDays](#16-testing-dr-chaos-engineering--gamedays)
17. [Real-World DR Scenarios](#17-real-world-dr-scenarios)
18. [AI & New Trends in DR (2024–2025)](#18-ai--new-trends-in-dr-20242025)
19. [DR Interview Questions & Answers](#19-dr-interview-questions--answers)
20. [Quick Reference Cheat Sheet](#20-quick-reference-cheat-sheet)

---

## 1. What Is Disaster Recovery?

**Disaster Recovery (DR)** is the set of policies, tools, and procedures that enable an organisation to resume critical business functions after a disruptive event — whether that's a full AWS region failure, a ransomware attack, accidental mass deletion, a data centre fire, or a botched deployment that corrupts a production database.

DR is **not** the same as high availability (HA) or backup, though all three are often confused. They address different failure scopes:

```
Scope of Protection:
                    │ Component  │ AZ       │ Region   │ Global / Account
────────────────────┼────────────┼──────────┼──────────┼──────────────────
High Availability   │     ✓      │    ✓     │          │
Disaster Recovery   │     ✓      │    ✓     │    ✓     │       ✓
Backup              │     ✓      │    ✓     │    ✓     │       ✓
```

### What Counts as a "Disaster"?

| Category | Examples |
|---|---|
| **Infrastructure failure** | Full AWS region outage, AZ power loss, hardware failure |
| **Human error** | `DROP TABLE` on production, mass delete of S3 objects, wrong Terraform destroy |
| **Cyber attack** | Ransomware encrypting all EBS volumes, account compromise, DDoS |
| **Application failure** | Bad deployment corrupts data model, runaway process corrupts DB |
| **Natural disaster** | Earthquake/flood affecting a data centre or AWS region |
| **Dependency failure** | Third-party DNS provider goes down, CDN outage, payment gateway failure |
| **Compliance-triggered** | Regulatory order requiring immediate data isolation or deletion |

### Why DR Matters for DevOps Engineers

In traditional IT, DR was the DBA's and SysAdmin's domain. In modern cloud-native engineering, DR is **owned by the DevOps/SRE team** because:
- Infrastructure is code — DR environments are Terraform modules
- Failover is automated — CloudWatch alarms trigger Lambda or Systems Manager runbooks
- DR is tested in CI/CD — chaos engineering is part of the pipeline
- Recovery playbooks are stored in Git — versioned, reviewed, executable

If you cannot answer "what is your RTO/RPO?" for your production system, you do not have a DR strategy — you have wishful thinking.

---

## 2. Core DR Metrics: RTO, RPO, and Beyond

These metrics define the business requirements that your DR architecture must satisfy. Every architectural decision maps back to these numbers.

### RTO — Recovery Time Objective

> **How long can the business tolerate the system being unavailable?**

RTO is the maximum acceptable time between a disaster declaration and the system being fully operational again.

```
Disaster occurs at T=0
  │
  ├── T+0 to T+X:  Detection, alerting, declaration
  ├── T+X to T+Y:  Failover execution (automated or manual)
  └── T+Y:         System fully operational in DR environment

RTO = T+Y  (must be ≤ business requirement)
```

| RTO Target | Architecture Required | Typical Cost |
|---|---|---|
| < 1 minute | Active-Active Multi-Site | Very high |
| 1–15 minutes | Warm Standby + automated failover | High |
| 15–60 minutes | Pilot Light + pre-provisioned infra | Medium |
| Hours | Backup & Restore | Low |
| Days | Cold standby / manual restore | Minimal |

### RPO — Recovery Point Objective

> **How much data loss can the business tolerate?**

RPO is the maximum age of data that must be recovered. If RPO = 1 hour, the business accepts losing up to 1 hour of transactions.

```
Last good backup at T-60min
  │
  ├── T-60min to T-0: Ongoing transactions (lost if disaster hits here)
  └── T-0: Disaster

RPO = 60 minutes (1 hour of data loss accepted)
```

| RPO Target | Mechanism | Service |
|---|---|---|
| Near-zero (seconds) | Synchronous replication | Aurora Global, DynamoDB Global Tables |
| < 1 minute | Continuous async replication | AWS Elastic DR, RDS cross-region replica |
| 5–15 minutes | Incremental snapshots | AWS Backup continuous, DMS CDC |
| 1 hour | Hourly snapshots | Aurora automated backups |
| 24 hours | Daily snapshots | RDS automated backups (default) |

### Additional DR Metrics

| Metric | Definition | Relevance |
|---|---|---|
| **MTTR** (Mean Time to Recover) | Average time to restore service after a failure | Measures DR efficiency; tracked over time |
| **MTBF** (Mean Time Between Failures) | Average time between incidents | Measures system reliability |
| **RLO** (Recovery Level Objective) | What % of full capacity must be available at recovery | e.g., "70% capacity acceptable for first 4 hours" |
| **WRT** (Work Recovery Time) | Time to verify data integrity after restoration | Often forgotten in RTO calculations |
| **MTD** (Maximum Tolerable Downtime) | Absolute maximum outage before existential business impact | RTO must always be < MTD |

> **Critical insight:** RTO + WRT ≤ MTD. Many engineers calculate RTO but forget WRT — the time to validate that the recovered system's data is actually correct. A 30-minute restore that takes 2 hours to validate has an actual recovery time of 2.5 hours.

---

## 3. DR vs. Backup vs. High Availability

These three are often conflated. They are complementary but serve different purposes:

### High Availability (HA)

**Goal:** Eliminate single points of failure so individual component failures cause no downtime.
**Scope:** Component and AZ level.
**Mechanism:** Redundancy — multiple instances, Multi-AZ deployment, health checks + automatic replacement.
**Examples:**
- RDS Multi-AZ: synchronous standby in second AZ, automatic failover in 60 seconds
- ALB across multiple AZs
- EKS across 3 AZs with pod anti-affinity rules
- S3: 11 nines durability (11×9s), data stored across 3+ AZs

```
HA Architecture (single region, multi-AZ):
  AZ-a: EC2 app server + RDS primary
    ↕ sync replication
  AZ-b: EC2 app server + RDS standby  ← fails over automatically
  AZ-c: EC2 app server
        ↑
    ALB distributes across all 3 AZs
```

HA does **not** protect against: region-level outages, data corruption/deletion (corrupts all copies simultaneously), ransomware, account-level compromise.

### Backup

**Goal:** Create point-in-time copies of data that can be restored after data loss or corruption.
**Scope:** Data integrity — restoring to a known-good state.
**Mechanism:** Snapshots (EBS, RDS, EFS), S3 versioning, AWS Backup policies.
**Key property:** A backup is **immutable** — ransomware cannot reach it if stored in a separate account with Object Lock (WORM).

Backup alone does **not** guarantee fast RTO — restoring a 10TB database from backup can take hours.

### Disaster Recovery

**Goal:** Restore full business operations in an alternate location after a catastrophic event that HA and backup cannot handle alone.
**Scope:** Entire application stack — compute, database, network, DNS.
**Mechanism:** Combination of replication (for low RPO), pre-provisioned infrastructure (for low RTO), and automated failover (for speed).

```
The complete picture:
  HA        → prevents most failures from causing downtime
  Backup    → recovers data after corruption or deletion
  DR        → restores entire systems after catastrophic failure

All three are required. None replaces the others.
```

---

## 4. The Four AWS DR Strategies

AWS defines four DR strategies in its Well-Architected Framework, ordered from cheapest/slowest to most expensive/fastest:

```
Cost ──────────────────────────────────────────────────────────────────▶
     Backup &    Pilot       Warm          Multi-Site
     Restore     Light       Standby       Active-Active
RTO  Hours      10s min     Minutes       Seconds/Zero
RPO  Hours      Minutes     Seconds/Min   Near-Zero
```

### Strategy 1 — Backup and Restore

**Concept:** No pre-provisioned infrastructure in the DR region. Data is backed up to S3 or cross-region snapshots. In a disaster, you restore from backup and rebuild the infrastructure from IaC (Terraform).

**RTO:** 1–8 hours (time to restore data + rebuild infrastructure)
**RPO:** Up to 24 hours (depends on backup frequency)
**Cost:** Cheapest — only storage costs for backups

**Architecture:**
```
Primary Region (eu-central-1)    DR Region (eu-west-1)
─────────────────────────────    ──────────────────────────────
EC2 + RDS + EKS                  Nothing running
    │
    ├── EBS snapshots ───────────────────────────────────▶ S3 (CRR)
    ├── RDS snapshots ──────────────────────────────────▶ DR region copy
    └── S3 objects ─────────────────────────────────────▶ S3 CRR bucket

Disaster → IaC (Terraform) rebuilds infra from scratch
         → Restore DB from latest snapshot
         → Restore app from AMI or container images in ECR
```

**When to use:** Non-critical workloads, development environments, cost-constrained organisations, applications where hours of downtime are acceptable.

**DevOps action:** Ensure Terraform code and all AMIs/container images are available in the DR region. Test the restore procedure regularly — "untested backups are not backups."

### Strategy 2 — Pilot Light

**Concept:** The minimum core infrastructure is always running in the DR region — typically just the database (replicated) and a skeleton of the application. Compute is scaled to zero (or minimal). In a disaster, you "ignite" the pilot light by scaling up compute and switching traffic.

**RTO:** 10–30 minutes (time to scale up EC2/ECS/EKS nodes)
**RPO:** Minutes (continuous DB replication)
**Cost:** Low to medium — replication costs + minimal compute in DR region

**Architecture:**
```
Primary Region (eu-central-1)         DR Region (eu-west-1)
─────────────────────────────         ──────────────────────────────────
EKS cluster (full)                    EKS cluster (0 nodes — control plane only)
RDS Aurora Primary ────CDC──────────▶ RDS Aurora Read Replica (promotable)
S3 buckets ─────────────CRR─────────▶ S3 DR buckets
ECR images ─────────────replication──▶ ECR DR registry
Route 53: primary region              Route 53: failover record (inactive)

Disaster:
1. Promote Aurora replica → standalone primary
2. Scale EKS node group from 0 → N nodes (5-10 min)
3. Apply Helm charts / Argo CD sync
4. Update Route 53 failover record → DR region
```

**DevOps action:** Automate the scale-up via Systems Manager Automation document or Lambda. Target: scale-up script runs in < 10 minutes.

### Strategy 3 — Warm Standby

**Concept:** A scaled-down but fully functional copy of the production environment runs continuously in the DR region. At disaster, you scale it up to production capacity and switch traffic.

**RTO:** 2–15 minutes (time to scale up from minimum to full capacity)
**RPO:** Seconds (continuous real-time replication)
**Cost:** Medium to high — running reduced-capacity infrastructure 24/7

**Architecture:**
```
Primary Region (eu-central-1)         DR Region (eu-west-1)
─────────────────────────────         ──────────────────────────────────
EKS: 10 nodes (prod capacity)  ──▶   EKS: 2 nodes (minimum viable)
Aurora: db.r6g.large (primary) ──▶   Aurora: db.r6g.medium (read replica, same schema)
ALB → Route 53 (100% traffic)        ALB → Route 53 (0% traffic, failover record)
4 microservices, 3 replicas each      4 microservices, 1 replica each (warm)

Disaster:
1. Promote Aurora replica (2 min)
2. Scale EKS to production capacity (5 min)
3. Scale microservice replicas to production (2 min)
4. Route 53 health check detects primary down → auto-switch (1 min)
RTO ≈ 10 minutes
```

**When to use:** Business-critical applications where hours of RTO is unacceptable but the cost of active-active is too high.

### Strategy 4 — Multi-Site Active-Active

**Concept:** Full production capacity runs in two (or more) regions simultaneously, both serving live traffic. No failover needed — traffic is load-balanced between regions. If one region fails, the other handles 100% of traffic automatically.

**RTO:** Near-zero (seconds, from health check detection)
**RPO:** Near-zero (synchronous or sub-second replication)
**Cost:** Highest — full production infrastructure in multiple regions × 24/7

**Architecture:**
```
Region A (eu-central-1)              Region B (eu-west-1)
──────────────────────               ──────────────────────
EKS cluster (full prod)              EKS cluster (full prod)
DynamoDB Global Table ◀──────────▶  DynamoDB Global Table  ← bidirectional
Aurora Global Primary  ────────────▶ Aurora Global Secondary (< 1sec lag)
S3 (CRR both directions)◀──────────▶ S3 (CRR both directions)
ALB in Region A                      ALB in Region B
        │                                    │
        └─────── Route 53 / Global ──────────┘
                 Accelerator
                 (latency routing or
                  weighted 50/50)
```

**Use cases:** E-commerce (Black Friday), financial trading platforms, healthcare records, any system where seconds of downtime = significant revenue loss or safety risk.

**Database challenge:** The hardest part of active-active is the database. Write conflicts between regions must be resolved. DynamoDB Global Tables handles this with last-write-wins. Aurora Global allows only one writer region — the other region is read-only until failover. For true active-active writes, DynamoDB is the better choice.

---

## 5. AWS Elastic Disaster Recovery (EDR/DRS)

**AWS Elastic Disaster Recovery** (previously CloudEndure Disaster Recovery) is AWS's managed, continuous-replication DR service. It is the **primary tool for compute-level DR** — the DR equivalent of MGN for migration.

### How EDR Works

```
Source (on-prem or AWS primary region)    DR Region
─────────────────────────────────────     ──────────────────────────────
Server (EDR agent installed)              EDR Replication Server (auto-provisioned)
OS + data ────────────────TLS────────────▶ Staging EBS volumes (1:1 copy)
                          continuous       (kept hot, encrypted at rest)
                          block-level
                          replication

Disaster:
  1. Declare DR event in EDR console or via API
  2. EDR launches Recovery Instances from staging EBS volumes
  3. Instances boot and applications start
  4. DNS/load balancer switched to recovery instances
  RTO: typically < 15 minutes
  RPO: seconds (continuous replication)
```

### EDR vs. MGN

| Feature | AWS EDR (Disaster Recovery) | AWS MGN (Migration) |
|---|---|---|
| **Purpose** | Ongoing DR — protect production | One-time migration |
| **Replication** | Continuous, ongoing | Continuous until cutover, then stops |
| **Source** | On-prem OR another AWS region | On-prem or other cloud |
| **Recovery** | Launch recovery instances on demand | Cutover instances become permanent |
| **Cost model** | Per protected server per hour ($0.028/hr) | Per replicated server per hour |
| **Post-recovery** | Can fail back to source | No failback (migration is one-way) |
| **Use case** | DR for existing production | Moving workloads to AWS |

### EDR CLI Commands

```bash
# Install EDR agent on Linux source server
wget -O ./aws-replication-installer-init.py \
  https://aws-elastic-disaster-recovery-us-east-1.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py

sudo python3 aws-replication-installer-init.py \
  --region eu-west-1 \
  --aws-access-key-id AKIAIOSFODNN7EXAMPLE \
  --aws-secret-access-key wJalrXUtnFEMI/K7MDENG \
  --no-prompt

# List all protected source servers
aws drs describe-source-servers \
  --query 'items[*].{
    ID:sourceServerID,
    Hostname:sourceProperties.identificationHints.hostname,
    State:dataReplicationInfo.dataReplicationState,
    LastSeen:dataReplicationInfo.dataReplicationTimestamps.lastHeartbeat,
    LagMinutes:dataReplicationInfo.dataReplicationTimestamps.eTA
  }' \
  --output table

# Check replication lag for a specific server
aws drs describe-source-servers \
  --filters sourceServerIDs=s-1234567890abcdef0 \
  --query 'items[0].dataReplicationInfo.{
    State:dataReplicationState,
    LagDuration:lagDuration,
    LastSync:dataReplicationTimestamps.lastHeartbeat
  }'

# ---- FAILOVER PROCEDURE ----

# Step 1: Start recovery instances (launches DR instances from staging volumes)
aws drs start-recovery \
  --source-servers '[{"sourceServerID":"s-1234567890abcdef0"}]' \
  --is-drill false  # true for DR drills, false for real failover

# Step 2: Describe recovery instances (get instance IDs)
aws drs describe-recovery-instances \
  --query 'items[*].{
    RecoveryID:recoveryInstanceID,
    SourceID:sourceServerID,
    EC2InstanceID:recoveryInstanceProperties.identificationHints.awsInstanceID,
    State:dataReplicationInfo.dataReplicationState
  }'

# Step 3: After validating recovery instances — stop failback replication
aws drs stop-failback \
  --recovery-instance-id ri-1234567890abcdef0

# ---- DR DRILL (non-destructive test) ----
aws drs start-recovery \
  --source-servers '[{"sourceServerID":"s-1234567890abcdef0"}]' \
  --is-drill true   # Drill mode — won't affect production

# Terminate drill instances after validation
aws drs terminate-recovery-instances \
  --recovery-instance-ids '["ri-1234567890abcdef0"]'
```

### EDR Launch Settings (Pre-configured for fast recovery)

```bash
# Configure launch settings for a source server
aws drs update-launch-configuration \
  --source-server-id s-1234567890abcdef0 \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method BASIC \
  --copy-tags true \
  --licensing '{"osByol":false}' \
  --boot-mode USE_SOURCE
```

---

## 6. Database Disaster Recovery

Databases are the hardest component to protect in DR. Data loss is the most catastrophic failure mode — and the database is almost always on the critical path for RTO.

### RDS Multi-AZ — AZ-Level HA (Not Regional DR)

Multi-AZ provides automatic failover within a region when the primary DB instance fails. It is HA, not DR — both AZs are in the same region, so a regional disaster affects both.

```
PRIMARY REGION (eu-central-1)
  AZ-a: RDS Primary (read + write)
    │
    │ synchronous replication
    │ (primary waits for standby ACK before committing)
    │
  AZ-b: RDS Standby (hot, not readable)

  Failover triggers:
    - AZ-a hardware failure
    - DB instance crash
    - Network partition to AZ-a
    - Manual failover (for maintenance)

  Failover time: 60–120 seconds (DNS CNAME updated)
  Data loss: Zero (synchronous replication)
```

### Aurora Global Database — Regional DR

Aurora Global is the correct tool for **cross-region disaster recovery** with RDS-compatible databases.

```
Primary Region (eu-central-1)           DR Region (eu-west-1)
─────────────────────────────           ──────────────────────────────
Aurora Writer (read + write)            Aurora Secondary Cluster (read-only)
    │                                       │
    └──── Dedicated replication ────────────┘
          network (not public internet)
          Typical lag: < 1 second
          RPO: < 1 second

Failover (planned or unplanned):
  aws rds failover-global-cluster \
    --global-cluster-identifier my-global-cluster \
    --target-db-cluster-identifier arn:aws:rds:eu-west-1:...:cluster:aurora-dr

  RTO: < 1 minute (managed failover)
  DNS: Cluster endpoints update automatically
```

```bash
# Create Aurora Global Database
aws rds create-global-cluster \
  --global-cluster-identifier my-global-aurora \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.05.2"

# Add primary cluster to global cluster
aws rds modify-db-cluster \
  --db-cluster-identifier aurora-primary \
  --global-cluster-identifier my-global-aurora

# Create secondary cluster in DR region
aws rds create-db-cluster \
  --db-cluster-identifier aurora-dr-secondary \
  --engine aurora-mysql \
  --engine-version "8.0.mysql_aurora.3.05.2" \
  --global-cluster-identifier my-global-aurora \
  --db-subnet-group-name dr-subnet-group \
  --vpc-security-group-ids sg-dr-12345678 \
  --region eu-west-1

# Create reader instance in DR region
aws rds create-db-instance \
  --db-instance-identifier aurora-dr-reader \
  --db-cluster-identifier aurora-dr-secondary \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --region eu-west-1

# Monitor global cluster replication lag
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=aurora-dr-secondary \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Maximum --region eu-west-1

# Failover to DR region (managed failover — < 1 minute)
aws rds failover-global-cluster \
  --global-cluster-identifier my-global-aurora \
  --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:aurora-dr-secondary

# Check new writer endpoint after failover
aws rds describe-db-clusters \
  --db-cluster-identifier aurora-dr-secondary \
  --region eu-west-1 \
  --query 'DBClusters[0].{Endpoint:Endpoint,Status:Status,Role:DBClusterMembers[?IsClusterWriter==`true`].DBInstanceIdentifier}'
```

### DynamoDB Global Tables — Active-Active Multi-Region

DynamoDB Global Tables provides **bidirectional, multi-master replication** across regions — true active-active with sub-second replication.

```
Region A (eu-central-1)    Region B (eu-west-1)    Region C (us-east-1)
────────────────────────    ──────────────────      ──────────────────
DynamoDB Table (R+W) ◀────▶ DynamoDB Table (R+W) ◀──▶ DynamoDB Table (R+W)
                         sub-second replication
                         last-write-wins conflict resolution

Route 53 latency routing:
  EU users → eu-central-1 (lowest latency)
  US users → us-east-1 (lowest latency)

AZ/Region failure → Route 53 health check fails
                 → traffic auto-routes to next region
                 → RTO: < 1 minute (DNS TTL)
                 → RPO: sub-second
```

```bash
# Create DynamoDB Global Table
aws dynamodb create-table \
  --table-name UserSessions \
  --attribute-definitions AttributeName=sessionId,AttributeType=S \
  --key-schema AttributeName=sessionId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --region eu-central-1

# Add replica in DR region
aws dynamodb update-table \
  --table-name UserSessions \
  --replica-updates '[
    {"Create":{"RegionName":"eu-west-1"}},
    {"Create":{"RegionName":"us-east-1"}}
  ]' \
  --region eu-central-1

# Check replication status
aws dynamodb describe-table \
  --table-name UserSessions \
  --region eu-central-1 \
  --query 'Table.Replicas[*].{Region:RegionName,Status:ReplicaStatus}'
```

### RDS Cross-Region Snapshots — Backup-and-Restore DR

For systems with less stringent RPO requirements (hours), cross-region snapshot copies provide DR at minimal cost.

```bash
# Automate cross-region snapshot copy with Lambda + EventBridge
# EventBridge rule triggers when an RDS automated snapshot completes

# Manual snapshot copy to DR region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:eu-central-1:123456789012:snapshot:rds:mydb-2024-01-15-02-00 \
  --target-db-snapshot-identifier mydb-dr-2024-01-15 \
  --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/mrk-dr-key \
  --copy-tags \
  --region eu-west-1

# Restore from snapshot in DR region
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-dr-restored \
  --db-snapshot-identifier mydb-dr-2024-01-15 \
  --db-instance-class db.r6g.large \
  --db-subnet-group-name dr-subnet-group \
  --vpc-security-group-ids sg-dr-12345678 \
  --region eu-west-1

# Wait for instance to be available
aws rds wait db-instance-available \
  --db-instance-identifier mydb-dr-restored \
  --region eu-west-1
```

### Database DR Decision Matrix

| Scenario | RPO Requirement | Recommended Solution | RTO |
|---|---|---|---|
| AZ failure only | Zero | RDS Multi-AZ (always on) | 60–120s |
| Regional DR, MySQL/PG | < 1 second | Aurora Global Database | < 1 min |
| Regional DR, Key-Value | Near-zero, active-active | DynamoDB Global Tables | < 1 min |
| Regional DR, SQL Server | Minutes | Cross-region read replica + promote | 10–30 min |
| Regional DR, Oracle | Minutes–hours | DMS CDC to Aurora + SCT | Hours |
| Low-cost DR, any engine | Hours | Cross-region snapshot copy | Hours |
| On-prem DB to AWS DR | Minutes | DMS CDC to RDS (ongoing) | 10–30 min |

---

## 7. Kubernetes & Container DR (EKS + Velero)

Kubernetes DR requires protecting two layers: the **control plane configuration** (manifests, Helm releases, ConfigMaps, Secrets) and the **persistent data** (PersistentVolumes).

### DR Architecture for EKS

```
Primary Region (eu-central-1)         DR Region (eu-west-1)
─────────────────────────────         ──────────────────────────────
EKS Cluster A (active)                EKS Cluster B
  ├── Production workloads             ├── Warm standby (scaled down or 0)
  ├── Persistent volumes (EFS)         ├── EFS replicated (AWS DataSync)
  └── Helm releases (Argo CD)         └── Argo CD synced to same Git repo

Git Repository (source of truth):
  ├── Infrastructure (Terraform)
  └── Application manifests (Helm values, Argo CD apps)

Velero S3 bucket ──────────CRR──────▶ Velero DR S3 bucket
(backup of cluster state)              (available in DR region)

Failover:
  1. Restore Velero backup to DR cluster
  2. Argo CD syncs from Git
  3. Route 53 / Global Accelerator switches traffic
  RTO: 15–30 minutes
```

### Velero — Kubernetes Backup and DR

Velero is the de facto standard for Kubernetes backup, restore, and migration. It backs up:
- All Kubernetes objects (Deployments, Services, ConfigMaps, Secrets, PVCs, RBAC, CRDs)
- PersistentVolume data (using file-system backup or CSI snapshots)

```bash
# Install Velero with AWS plugin on primary EKS cluster
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups-primary \
  --backup-location-config region=eu-central-1 \
  --snapshot-location-config region=eu-central-1 \
  --use-node-agent \
  --default-volumes-to-fs-backup \
  --secret-file ./credentials-velero

# Configure backup storage location for DR region
velero backup-location create dr-region \
  --provider aws \
  --bucket velero-backups-dr \
  --config region=eu-west-1

# Create scheduled backup (every 6 hours)
velero schedule create production-dr \
  --schedule "0 */6 * * *" \
  --include-namespaces production,monitoring,ingress-nginx \
  --storage-location dr-region \
  --volume-snapshot-locations dr-region \
  --ttl 168h  # Keep backups for 7 days

# Create immediate backup before a risky deployment
velero backup create pre-deployment-$(date +%Y%m%d-%H%M) \
  --include-namespaces production \
  --storage-location dr-region \
  --wait

# List available backups
velero backup get

# Describe a backup (shows what was captured)
velero backup describe production-dr-20240115180000 --details

# ---- RESTORE (in DR region EKS cluster) ----

# Install Velero on DR cluster (same config, pointing to DR S3 bucket)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero-backups-dr \
  --backup-location-config region=eu-west-1 \
  --snapshot-location-config region=eu-west-1 \
  --use-node-agent \
  --secret-file ./credentials-velero

# Sync backups from S3 (so DR cluster sees primary's backups)
velero backup-location get  # Confirm backup location is available

# Restore from the latest backup
velero restore create production-restore \
  --from-backup production-dr-20240115180000 \
  --include-namespaces production \
  --restore-volumes true \
  --wait

# Check restore status
velero restore describe production-restore
velero restore logs production-restore

# Validate pods are running after restore
kubectl get pods -n production
kubectl get pvc -n production  # Check PVCs are bound
```

### Multi-Cluster GitOps with Argo CD for DR

The most robust EKS DR uses Argo CD in multi-cluster mode where **both the primary and DR cluster are always synced to the same Git repository**. The DR cluster runs at zero or minimum replicas.

```yaml
# Argo CD ApplicationSet — deploys to both primary and DR cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: production-app-multicluster
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: primary
        url: https://primary-eks-api.eu-central-1.example.com
        namespace: production
        replicaCount: "3"
      - cluster: dr
        url: https://dr-eks-api.eu-west-1.example.com
        namespace: production
        replicaCount: "1"  # Warm standby — 1 replica each service
  template:
    metadata:
      name: '{{cluster}}-production-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/k8s-manifests.git
        targetRevision: main
        path: apps/production
        helm:
          parameters:
          - name: replicaCount
            value: '{{replicaCount}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Persistent Volume DR with EFS Cross-Region Replication

```bash
# Enable EFS replication to DR region
aws efs create-replication-configuration \
  --source-file-system-id fs-primary-12345678 \
  --destinations '[{
    "Region": "eu-west-1",
    "KmsKeyId": "arn:aws:kms:eu-west-1:123456789012:key/mrk-dr-key"
  }]'

# Check replication status
aws efs describe-replication-configurations \
  --source-file-system-id fs-primary-12345678 \
  --query 'Replications[0].{
    State:Destinations[0].Status,
    LastSync:Destinations[0].LastReplicatedTimestamp,
    Destination:Destinations[0].FileSystemId
  }'

# RPO for EFS replication: typically < 15 minutes
# In a DR scenario, mount the replicated EFS in the DR EKS cluster
```

---

## 8. Storage DR (S3, EFS, EBS)

### S3 Cross-Region Replication (CRR)

S3 CRR asynchronously replicates objects from a source bucket to a destination bucket in another region. RPO is typically seconds to minutes.

```bash
# Enable CRR with Terraform
# (via Terraform — see Section 14 for full IaC)

# Check replication status for a specific object
aws s3api head-object \
  --bucket my-dr-bucket \
  --key path/to/object.dat \
  --region eu-west-1 \
  --query 'ReplicationStatus'
# Values: REPLICA (successfully replicated), PENDING, FAILED, COMPLETE

# List replication failures (check metrics)
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name OperationsFailedReplication \
  --dimensions Name=SourceBucket,Value=my-primary-bucket Name=RuleId,Value=entire-bucket \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum
```

### S3 Object Lock for Immutable Backups (Ransomware Protection)

```bash
# Create bucket with Object Lock (cannot be disabled after creation)
aws s3api create-bucket \
  --bucket company-immutable-backups \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

aws s3api put-object-lock-configuration \
  --bucket company-immutable-backups \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "GOVERNANCE",
        "Days": 90
      }
    }
  }'
# GOVERNANCE mode: can be overridden by admin with special IAM permission
# COMPLIANCE mode: CANNOT be overridden by ANYONE, including root — use for strict compliance
```

### EBS Snapshot DR

```bash
# Copy EBS snapshot to DR region (for Pilot Light / Backup-Restore)
aws ec2 copy-snapshot \
  --source-region eu-central-1 \
  --source-snapshot-id snap-1234567890abcdef0 \
  --description "DR copy of prod volume - $(date +%Y-%m-%d)" \
  --encrypted \
  --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/mrk-dr-key \
  --region eu-west-1

# Automate EBS snapshot cross-region copy using DLM (Data Lifecycle Manager)
aws dlm create-lifecycle-policy \
  --description "Daily EBS DR snapshot policy" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/DLMServiceRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["INSTANCE"],
    "TargetTags": [{"Key":"DR","Value":"true"}],
    "Schedules": [{
      "Name": "Daily snapshots with DR copy",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["03:00"]},
      "RetainRule": {"Count": 7},
      "CrossRegionCopyRules": [{
        "TargetRegion": "eu-west-1",
        "Encrypted": true,
        "CmkArn": "arn:aws:kms:eu-west-1:123456789012:key/mrk-dr-key",
        "RetainRule": {"Interval": 7, "IntervalUnit": "DAYS"},
        "CopyTags": true
      }]
    }]
  }'
```

---

## 9. Network-Level DR (Route 53, Global Accelerator)

Traffic routing is the **last mile of DR** — even if your compute and data are ready in the DR region, users won't reach it until DNS or anycast routing is updated.

### Route 53 Health Checks + Failover Routing

Route 53 health checks monitor your primary endpoint and automatically switch DNS to the DR endpoint when the primary is unhealthy.

```bash
# Create health check for primary ALB endpoint
aws route53 create-health-check \
  --caller-reference "primary-alb-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "app.company.com",
    "Port": 443,
    "ResourcePath": "/health",
    "RequestInterval": 10,
    "FailureThreshold": 3,
    "EnableSNI": true,
    "Regions": ["eu-west-1", "eu-central-1", "us-east-1"]
  }'

# Create primary DNS record with failover routing
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "app.company.com",
          "Type": "A",
          "SetIdentifier": "primary-eu-central-1",
          "Failover": "PRIMARY",
          "HealthCheckId": "abc12345-1234-1234-1234-abcdef123456",
          "TTL": 60,
          "AliasTarget": {
            "HostedZoneId": "Z215JYRZR1TBD5",
            "DNSName": "primary-alb.eu-central-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "app.company.com",
          "Type": "A",
          "SetIdentifier": "dr-eu-west-1",
          "Failover": "SECONDARY",
          "TTL": 60,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "dr-alb.eu-west-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

### AWS Global Accelerator — Faster Failover Than DNS

Route 53 failover is limited by DNS TTL (minimum 60 seconds, plus client-side caching). **AWS Global Accelerator** uses **anycast IPs** — the same two static IP addresses always point to your application, and Global Accelerator routes traffic to the nearest healthy endpoint in seconds (not minutes).

```
Without Global Accelerator (DNS-based failover):
  User → DNS lookup → waits for TTL expiry → new endpoint
  Failover time: 60s (TTL) + DNS propagation = 2–5 minutes

With Global Accelerator (anycast failover):
  User → anycast IP (always the same) → Global Accelerator → nearest healthy endpoint
  Failover time: < 30 seconds (health check → re-routing)
  Also: 60% better TCP connection establishment (AWS global backbone)
```

```bash
# Create Global Accelerator
aws globalaccelerator create-accelerator \
  --name production-accelerator \
  --ip-address-type IPV4 \
  --enabled \
  --region us-east-1  # Global Accelerator is global; managed from us-east-1

# Create listener (ports 80 + 443)
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/abcd1234 \
  --port-ranges FromPort=443,ToPort=443 FromPort=80,ToPort=80 \
  --protocol TCP \
  --region us-east-1

# Add endpoint groups (primary + DR regions)
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/abcd1234/listener/xyz789 \
  --endpoint-group-region eu-central-1 \
  --traffic-dial-percentage 100 \
  --health-check-path /health \
  --health-check-interval-seconds 10 \
  --threshold-count 3 \
  --endpoints EndpointId=arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/primary-alb/xxx,Weight=100,ClientIPPreservationEnabled=true \
  --region us-east-1

aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/abcd1234/listener/xyz789 \
  --endpoint-group-region eu-west-1 \
  --traffic-dial-percentage 0 \  # 0% normally — increases to 100% on failover
  --endpoints EndpointId=arn:aws:elasticloadbalancing:eu-west-1:123456789012:loadbalancer/app/dr-alb/yyy,Weight=100 \
  --region us-east-1

# During DR failover — shift traffic to DR region
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:accelerator/abcd1234/listener/xyz789/endpoint-group/eu-west-1 \
  --traffic-dial-percentage 100 \
  --region us-east-1

aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:accelerator/abcd1234/listener/xyz789/endpoint-group/eu-central-1 \
  --traffic-dial-percentage 0 \
  --region us-east-1
```

---

## 10. AWS Backup — Centralised Backup Management

**AWS Backup** is a centralised service that manages backup policies for EBS, RDS, Aurora, DynamoDB, EFS, FSx, EC2 (AMI), S3 (2023+), and VMware Cloud on AWS — across all regions and all accounts in your organisation.

### Key AWS Backup Concepts

| Concept | Definition |
|---|---|
| **Backup Plan** | Schedule and rules (frequency, retention, copy to DR region) |
| **Backup Vault** | Encrypted storage container for backups |
| **Recovery Point** | An individual backup (snapshot, copy) |
| **Vault Lock** | Object Lock for backup vaults — WORM, ransomware-proof |
| **Cross-Account Backup** | Copy backups to a separate AWS account (separate blast radius) |
| **AWS Backup Audit Manager** | Compliance reports proving backup policy adherence |

```bash
# Create a backup vault in the DR region
aws backup create-backup-vault \
  --backup-vault-name production-dr-vault \
  --encryption-key-arn arn:aws:kms:eu-west-1:123456789012:key/mrk-backup-key \
  --region eu-west-1

# Lock the vault — immutable backups for 7 years (compliance requirement)
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name production-dr-vault \
  --min-retention-days 1 \
  --max-retention-days 2555 \
  --changeable-for-days 3 \  # You have 3 days to change before lock is permanent
  --region eu-west-1

# Create backup plan (daily + weekly + monthly with cross-region copy)
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "production-dr-backup-plan",
    "Rules": [
      {
        "RuleName": "daily-backup",
        "TargetBackupVaultName": "production-primary-vault",
        "ScheduleExpression": "cron(0 3 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {"DeleteAfterDays": 7},
        "CopyActions": [{
          "DestinationBackupVaultArn": "arn:aws:backup:eu-west-1:123456789012:backup-vault:production-dr-vault",
          "Lifecycle": {"DeleteAfterDays": 30}
        }],
        "EnableContinuousBackup": true
      },
      {
        "RuleName": "weekly-backup",
        "TargetBackupVaultName": "production-primary-vault",
        "ScheduleExpression": "cron(0 5 ? * SUN *)",
        "Lifecycle": {"DeleteAfterDays": 90},
        "CopyActions": [{
          "DestinationBackupVaultArn": "arn:aws:backup:eu-west-1:123456789012:backup-vault:production-dr-vault",
          "Lifecycle": {"DeleteAfterDays": 365}
        }]
      }
    ]
  }'

# Assign resources to backup plan by tag
aws backup create-backup-selection \
  --backup-plan-id abc12345-1234-1234-1234-abcdef123456 \
  --backup-selection '{
    "SelectionName": "production-resources",
    "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupServiceRole",
    "ListOfTags": [
      {"ConditionType": "STRINGEQUALS", "ConditionKey": "Environment", "ConditionValue": "production"},
      {"ConditionType": "STRINGEQUALS", "ConditionKey": "DR", "ConditionValue": "true"}
    ]
  }'

# Restore from backup (RDS example)
aws backup start-restore-job \
  --recovery-point-arn arn:aws:backup:eu-west-1:123456789012:recovery-point/abcdef12-3456-7890 \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupServiceRole \
  --resource-type RDS \
  --metadata '{
    "DBInstanceIdentifier": "mydb-dr-restored",
    "DBSubnetGroupName": "dr-subnet-group",
    "VpcSecurityGroupIds": "sg-dr-12345678",
    "MultiAZ": "true",
    "DBInstanceClass": "db.r6g.large"
  }' \
  --region eu-west-1
```

---

## 11. DR in the Context of Cloud Migration

DR and cloud migration are deeply intertwined. Understanding this relationship is critical for DevOps engineers on migration projects.

### Pre-Migration: On-Premises DR (What You're Replacing)

Most organisations have poor on-premises DR:
- **DR site** = an expensive second data centre (colocation) running at 30% utilisation
- **Backup tapes** shipped weekly to off-site storage
- **DR tests** done once a year (if at all) — and often fail
- **RTO/RPO** targets defined on paper but never validated

The cloud migration is the opportunity to **massively improve DR** at lower cost.

### Using the Migration as a DR Improvement Programme

During the Mobilize phase of a migration, the Landing Zone already creates multi-region architecture. This is the right time to design DR in parallel:

```
Migration Phases                    DR Milestones
───────────────────                 ──────────────────────────────
Assess:                             Define RTO/RPO requirements per app
  └── App portfolio review          Classify apps by DR tier (1-4)

Mobilize:                           Build DR foundation
  ├── Landing Zone                  ├── Multi-account DR structure
  ├── Network (TGW, DX)             ├── Cross-region VPC + TGW
  ├── Security baseline             └── Backup vaults in DR region

Migrate Wave 1 (pilot):             Validate DR for pilot apps
  └── Non-critical apps             └── Backup & Restore tier

Migrate Wave 2-N (production):      Implement DR per app tier
  └── Production workloads          ├── Pilot Light or Warm Standby
                                    └── Active-Active for tier 1

Post-migration:                     DR testing programme
  └── Optimise                      └── Monthly DR drills, GameDays
```

### Using AWS DR Services as a Migration Transition Strategy

**EDR as a bridge:** Before migrating, install AWS Elastic Disaster Recovery agents on critical on-prem servers. This gives you:
1. Real-time replication to AWS (RPO = seconds)
2. Confidence that AWS can host the workload
3. A tested failover path before the actual migration cutover
4. In an emergency, you can fail over to AWS BEFORE the planned migration

```
Timeline:
  Week 1:  Install EDR agents on critical servers
  Week 2-4: Verify replication, run DR drill (is-drill=true)
  Week 5:  DR drill successful — confidence that AWS environment works
  Week 6:  Run actual migration cutover using MGN
         (now you know AWS can run it — EDR already proved it)
```

---

## 12. DevOps Engineering Responsibilities for DR

### Design Phase
- Define RTO/RPO per application tier (work with business stakeholders)
- Choose the appropriate DR strategy (cost vs. recovery time matrix)
- Design multi-region networking (TGW attachments, Route 53, Global Accelerator)
- Document the DR architecture with Mermaid diagrams in the runbook

### Build Phase (IaC)
- Write Terraform modules for DR infrastructure (see Section 14)
- Ensure all Terraform state is in S3 with cross-region replication (so IaC is recoverable)
- Configure S3 CRR for all data buckets
- Set up Aurora Global Database or DynamoDB Global Tables
- Install and configure EDR/Velero agents
- Configure AWS Backup plans and cross-account vault policies

### Operate Phase
- Monitor replication lag (CloudWatch alarms for `AuroraGlobalDBReplicationLag`, `ReplicaLag`)
- Monitor backup job success/failure (AWS Backup → EventBridge → SNS)
- Review and action Backup Audit Manager compliance reports monthly
- Rotate DR documentation — ensure runbooks reflect current architecture

### Test Phase (Critical — Most Teams Skip This)
- Run quarterly DR drills (EDR `is-drill=true`, Velero restore to DR cluster)
- Annual full failover test (actually switch production traffic to DR region)
- Document actual RTO/RPO achieved vs. targets — update architecture if targets missed
- Conduct chaos engineering experiments (see Section 16)

---

## 13. DevSecOps: Security During Failover

DR is a high-risk security moment. Attackers know that during a DR event, security controls are often relaxed "just to get things running." Maintain security discipline during failover.

### KMS Cross-Region Key Replication

All encrypted resources in the DR region need access to encryption keys. Use **KMS Multi-Region Keys (MRKs)** which are replicated across regions:

```bash
# Create primary MRK in eu-central-1
aws kms create-key \
  --description "Production DR master key" \
  --multi-region true \
  --region eu-central-1

# Replicate to DR region
aws kms replicate-key \
  --key-id arn:aws:kms:eu-central-1:123456789012:key/mrk-abc123 \
  --replica-region eu-west-1 \
  --description "DR replica key eu-west-1" \
  --region eu-central-1

# Now both regions have the same key material
# Resources encrypted with mrk-abc123 in eu-central-1
# can be decrypted by the replica in eu-west-1 — no re-encryption needed
```

### IAM Roles Must Exist in DR Region Before Failover

IAM is global, but IAM policies referencing region-specific resources must work in both regions. Before a DR event, verify:

```bash
# Check that the EC2 instance profile works in DR region
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/AppInstanceRole \
  --role-session-name dr-validation

# Test that the role can access DR-region resources
aws s3 ls s3://my-dr-bucket/ --region eu-west-1

# Verify Secrets Manager secrets exist in DR region
aws secretsmanager describe-secret \
  --secret-id prod/myapp/database \
  --region eu-west-1
# If this fails — create/replicate the secret before a DR event, not during
```

### Secrets Replication for DR

```bash
# Replicate secrets to DR region using Lambda + EventBridge
# Or manually:
SOURCE_SECRET=$(aws secretsmanager get-secret-value \
  --secret-id prod/myapp/database \
  --query SecretString --output text \
  --region eu-central-1)

aws secretsmanager create-secret \
  --name prod/myapp/database \
  --secret-string "$SOURCE_SECRET" \
  --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/mrk-abc123 \
  --region eu-west-1

# Better: Use AWS Secrets Manager cross-region replication (2022+)
aws secretsmanager replicate-secret-to-regions \
  --secret-id prod/myapp/database \
  --add-replica-regions '[{"Region":"eu-west-1","KmsKeyId":"arn:aws:kms:eu-west-1:123456789012:key/mrk-abc123"}]'
```

### Maintain Security Controls During DR

Never disable security controls to speed up DR. The DR runbook must explicitly include:
- GuardDuty is enabled in DR region (verify before DR event)
- Security Hub is enabled in DR region
- CloudTrail is writing to the log archive account (org-level trail covers all regions automatically)
- WAF rules are replicated to DR ALB (or use the same WAF WebACL via CloudFront)
- No `0.0.0.0/0` ingress rules on security groups in DR environment
- MFA enforcement remains active for console access during DR

---

## 14. Terraform for DR Infrastructure

### Module Structure for DR

```hcl
# DR module directory structure
# modules/
#   ├── dr-networking/     VPC, subnets, TGW attachment, security groups in DR region
#   ├── dr-database/       Aurora Global secondary, DynamoDB replica, RDS snapshot copy
#   ├── dr-compute/        EKS (DR), EC2 AMI pre-baked, EDR configuration
#   ├── dr-storage/        S3 CRR rules, EFS replication, EBS DLM policies
#   └── dr-routing/        Route 53 health checks, failover records, Global Accelerator
```

```hcl
# DR Networking — VPC in DR region mirroring production
module "dr_vpc" {
  source = "./modules/dr-networking"
  providers = {
    aws = aws.dr_region  # Provider alias for DR region
  }

  vpc_cidr            = "10.10.0.0/16"  # Non-overlapping with primary (10.1.0.0/16)
  private_subnet_cidrs = ["10.10.1.0/24", "10.10.2.0/24", "10.10.3.0/24"]
  public_subnet_cidrs  = ["10.10.10.0/24", "10.10.11.0/24", "10.10.12.0/24"]
  azs                  = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  name_prefix          = "dr"
  transit_gateway_id   = aws_ec2_transit_gateway.dr.id
}

# Provider configuration for multi-region
provider "aws" {
  alias  = "primary"
  region = "eu-central-1"
}

provider "aws" {
  alias  = "dr_region"
  region = "eu-west-1"
}

# Aurora Global Database setup
resource "aws_rds_global_cluster" "main" {
  provider = aws.primary

  global_cluster_identifier = "prod-global-aurora"
  engine                    = "aurora-mysql"
  engine_version            = "8.0.mysql_aurora.3.05.2"
  database_name             = "appdb"
  storage_encrypted         = true
}

resource "aws_rds_cluster" "primary" {
  provider = aws.primary

  cluster_identifier        = "aurora-primary"
  engine                    = aws_rds_global_cluster.main.engine
  engine_version            = aws_rds_global_cluster.main.engine_version
  global_cluster_identifier = aws_rds_global_cluster.main.id
  master_username           = "admin"
  master_password           = var.db_master_password
  db_subnet_group_name      = aws_db_subnet_group.primary.name
  vpc_security_group_ids    = [aws_security_group.aurora_primary.id]
  storage_encrypted         = true
  kms_key_id                = aws_kms_key.primary.arn
  backup_retention_period   = 7
  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "aurora-primary-final"
}

resource "aws_rds_cluster" "dr_secondary" {
  provider = aws.dr_region

  cluster_identifier        = "aurora-dr-secondary"
  engine                    = aws_rds_global_cluster.main.engine
  engine_version            = aws_rds_global_cluster.main.engine_version
  global_cluster_identifier = aws_rds_global_cluster.main.id
  db_subnet_group_name      = aws_db_subnet_group.dr.name
  vpc_security_group_ids    = [aws_security_group.aurora_dr.id]
  storage_encrypted         = true
  kms_key_id                = aws_kms_key.dr.arn
  skip_final_snapshot       = true  # DR secondary — no final snapshot needed

  depends_on = [aws_rds_cluster.primary]
}

# S3 Cross-Region Replication
resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "company-app-data-primary"
}

resource "aws_s3_bucket_versioning" "primary" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket" "dr" {
  provider = aws.dr_region
  bucket   = "company-app-data-dr"
}

resource "aws_s3_bucket_versioning" "dr" {
  provider = aws.dr_region
  bucket   = aws_s3_bucket.dr.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_replication_configuration" "crr" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  role     = aws_iam_role.s3_replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"  # Cheaper — DR data accessed rarely

      encryption_configuration {
        replica_kms_key_id = aws_kms_key.dr.arn
      }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }
  }

  depends_on = [
    aws_s3_bucket_versioning.primary,
    aws_s3_bucket_versioning.dr
  ]
}

# Route 53 Health Check + Failover records
resource "aws_route53_health_check" "primary" {
  fqdn              = "app.company.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10
  regions           = ["eu-west-1", "eu-central-1", "us-east-1"]

  tags = { Name = "primary-health-check" }
}

resource "aws_route53_record" "primary_failover" {
  zone_id        = data.aws_route53_zone.main.zone_id
  name           = "app.company.com"
  type           = "A"
  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "dr_failover" {
  provider       = aws.dr_region
  zone_id        = data.aws_route53_zone.main.zone_id
  name           = "app.company.com"
  type           = "A"
  set_identifier = "dr"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

---

## 15. DR Runbooks and Automation

A DR runbook is a **precise, step-by-step operational document** for executing a failover. It must be:
- Written in advance (not during a disaster)
- Tested and validated
- Version-controlled in Git
- Executable by anyone on the on-call rotation (not just the author)
- Include rollback steps for every action

### DR Runbook Template Structure

```markdown
# DR Runbook: Full Region Failover (eu-central-1 → eu-west-1)

## Trigger Conditions
- Primary region health check failing for > 5 minutes
- AWS Service Health Dashboard shows eu-central-1 degradation
- PagerDuty P1 incident declared

## Pre-Failover Checklist (complete before starting)
- [ ] P1 incident declared in PagerDuty (incident-ID: ________)
- [ ] Incident commander assigned: _______________
- [ ] Stakeholders notified: CTO, Product Owner, Support Lead
- [ ] Current DRS replication lag < 5 minutes (check: CloudWatch / EDR console)
- [ ] DR Aurora replica lag < 30 seconds (check: CloudWatch AuroraGlobalDBReplicationLag)
- [ ] All team members on bridge call (Slack War Room: #dr-activation-YYYY-MM-DD)

## Failover Steps

### Step 1: Suspend Write Operations (if primary still partially available)
Time estimate: 2 minutes
Command:
  [application-specific command to enable read-only mode]
Validation: HTTP GET /api/health returns {"status":"read-only"}

### Step 2: Promote Aurora Global Secondary
Time estimate: 2-3 minutes
Command:
  aws rds failover-global-cluster \
    --global-cluster-identifier prod-global-aurora \
    --target-db-cluster-identifier aurora-dr-secondary \
    --region eu-central-1
Validation:
  aws rds describe-db-clusters \
    --db-cluster-identifier aurora-dr-secondary --region eu-west-1 \
    --query 'DBClusters[0].{Writer:DBClusterMembers[?IsClusterWriter==`true`]}'
  Expected: Writer exists in eu-west-1

### Step 3: Update Secrets Manager (DR DB endpoint)
Time estimate: 1 minute
Command:
  NEW_ENDPOINT=$(aws rds describe-db-clusters \
    --db-cluster-identifier aurora-dr-secondary --region eu-west-1 \
    --query 'DBClusters[0].Endpoint' --output text)
  aws secretsmanager update-secret \
    --secret-id prod/myapp/database \
    --secret-string "{\"host\":\"$NEW_ENDPOINT\",...}" \
    --region eu-west-1
Validation: Application connects to new DB endpoint

### Step 4: Scale EKS to Production Capacity
Time estimate: 5-8 minutes
Command:
  aws eks update-nodegroup-config \
    --cluster-name prod-dr-cluster \
    --nodegroup-name general-workers \
    --scaling-config desiredSize=10,minSize=5,maxSize=30 \
    --region eu-west-1
Validation: kubectl get nodes -n production (all nodes Ready)

### Step 5: Deploy Application (Argo CD Sync)
Time estimate: 3-5 minutes
Command:
  argocd app sync production-app --server dr-argocd.eu-west-1.example.com
Validation: All pods Running, Health: Healthy in Argo CD

### Step 6: Run Smoke Tests
Time estimate: 2 minutes
Command: ./scripts/smoke-test.sh --env dr
Validation: All 15 critical endpoints return 200

### Step 7: Switch Traffic to DR Region
Time estimate: 1 minute (DNS TTL-dependent)
Command:
  aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn arn:aws:globalaccelerator::...:eu-west-1 \
    --traffic-dial-percentage 100
  aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn arn:aws:globalaccelerator::...:eu-central-1 \
    --traffic-dial-percentage 0
Validation: curl -I https://app.company.com returns 200

## Total estimated RTO: 15-25 minutes

## Post-Failover Monitoring (first 30 minutes)
- Watch CloudWatch dashboard: MigrationValidation-DR
- Alarm threshold: 5xx > 2% triggers re-evaluation
- Database: monitor AuroraGlobalDBReplicationLag (should be N/A — now the writer)
- Inform support team: DR is active, primary region is unavailable

## Rollback Procedure (if DR failover fails)
1. Restore Global Accelerator: primary 100%, DR 0%
2. Promote Aurora primary back (if still available)
3. Scale EKS DR back to minimum
4. Declare all-clear in PagerDuty
```

### Automated DR with AWS Systems Manager Automation

```yaml
# SSM Automation Document — automates Steps 2-6 of the runbook
description: "Automated DR Failover - eu-central-1 to eu-west-1"
schemaVersion: "0.3"
assumeRole: "arn:aws:iam::123456789012:role/SSMAutomationRole"

mainSteps:
  - name: PromoteAuroraGlobal
    action: aws:executeAwsApi
    inputs:
      Service: rds
      Api: FailoverGlobalCluster
      GlobalClusterIdentifier: prod-global-aurora
      TargetDbClusterIdentifier: "arn:aws:rds:eu-west-1:123456789012:cluster:aurora-dr-secondary"
    outputs:
      - Name: status
        Selector: $.GlobalCluster.Status

  - name: WaitForAuroraPromotion
    action: aws:waitForAwsResourceProperty
    inputs:
      Service: rds
      Api: DescribeDBClusters
      DBClusterIdentifier: aurora-dr-secondary
      PropertySelector: "$.DBClusters[0].Status"
      DesiredValues: ["available"]
    timeoutSeconds: 300

  - name: ScaleEKSNodeGroup
    action: aws:executeAwsApi
    inputs:
      Service: eks
      Api: UpdateNodegroupConfig
      clusterName: prod-dr-cluster
      nodegroupName: general-workers
      scalingConfig:
        desiredSize: 10
        minSize: 5
        maxSize: 30

  - name: SwitchGlobalAccelerator
    action: aws:executeAwsApi
    inputs:
      Service: globalaccelerator
      Api: UpdateEndpointGroup
      EndpointGroupArn: "arn:aws:globalaccelerator::123456789012:accelerator/.../eu-west-1"
      TrafficDialPercentage: 100

  - name: SendNotification
    action: aws:publish
    inputs:
      TopicArn: "arn:aws:sns:us-east-1:123456789012:dr-notifications"
      Message: "DR failover completed. RTO achieved. Monitoring active."
```

---

## 16. Testing DR: Chaos Engineering & GameDays

> **Untested DR is not DR. It is a documented hope.**

The only way to know your RTO/RPO targets are achievable is to regularly test them under realistic conditions.

### Levels of DR Testing

| Level | Description | Frequency | Risk |
|---|---|---|---|
| **Plan review** | Review runbooks for accuracy | Monthly | None |
| **Tabletop exercise** | Walk through a scenario verbally with the team | Quarterly | None |
| **Component test** | Test one layer: restore a DB snapshot, failover Aurora | Monthly | Low |
| **DR drill (partial)** | Execute failover steps in test environment | Quarterly | Low |
| **Full DR test** | Simulate a real regional outage with actual production failover | Annually | Medium-High |
| **Chaos engineering** | Continuously inject failures in production to validate resilience | Continuous | Managed |

### AWS Fault Injection Service (FIS) — Chaos Engineering

AWS FIS is the managed chaos engineering service. It injects failures into your infrastructure in a controlled way.

```bash
# Create a chaos experiment: terminate 50% of EKS nodes in a random AZ
aws fis create-experiment-template \
  --description "Simulate AZ failure: terminate EKS nodes in eu-central-1a" \
  --stop-conditions '[{"source":"aws:cloudwatch:alarm","value":"arn:aws:cloudwatch:...:alarm:dr-critical-alarm"}]' \
  --targets '{
    "eksNodes": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": {"eks:nodegroup-name": "general-workers", "aws:availability-zone": "eu-central-1a"},
      "selectionMode": "PERCENT(50)"
    }
  }' \
  --actions '{
    "terminateNodes": {
      "actionId": "aws:ec2:terminate-instances",
      "description": "Terminate 50% of nodes in eu-central-1a",
      "targets": {"Instances": "eksNodes"}
    }
  }' \
  --role-arn arn:aws:iam::123456789012:role/FISRole \
  --log-configuration '{"cloudWatchLogsConfiguration":{"logGroupArn":"arn:aws:logs:eu-central-1:123456789012:log-group:/fis/experiments"},"logSchemaVersion":2}'

# Run the experiment
aws fis start-experiment \
  --experiment-template-id EXTabc123

# Watch the results
aws fis get-experiment \
  --id EXPabc123 \
  --query '{Status:state.status,Actions:actions}'
```

### Common Chaos Scenarios to Test

```bash
# Scenario 1: Kill the RDS primary — does Multi-AZ failover in < 2 minutes?
aws rds reboot-db-instance \
  --db-instance-identifier mydb-primary \
  --force-failover

# Scenario 2: Delete the Route 53 primary record — does failover routing activate?
# (test in staging only — never in production without a runbook)

# Scenario 3: Fill up a disk to 95% — does CloudWatch alarm fire before impact?
dd if=/dev/zero of=/tmp/fill_disk bs=1M count=50000

# Scenario 4: Block all traffic from on-prem — does hybrid connectivity alarm trigger?
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-12345678 \
  --remove-route-table-ids rtb-12345678  # Remove route (simulate connectivity loss)

# Scenario 5: Corrupt an S3 object — does S3 versioning allow recovery?
aws s3api put-object \
  --bucket my-app-data \
  --key critical/config.json \
  --body /dev/null  # Corrupt by replacing with empty file
# Recovery:
aws s3api list-object-versions \
  --bucket my-app-data --prefix critical/config.json \
  --query 'Versions[*].{Version:VersionId,Date:LastModified}'
aws s3api get-object \
  --bucket my-app-data --key critical/config.json \
  --version-id abc123previousVersion /tmp/recovered-config.json
```

### GameDay Structure

A **GameDay** is a structured exercise where the team simulates a real disaster:

```
GameDay Agenda (Half-Day Format):
  09:00 - 09:15  Brief: scenario description, rules, success criteria
  09:15 - 09:30  Preparation: confirm everyone has runbook access, tools open
  09:30 - 10:00  [SIMULATED DISASTER DECLARED]
                 - Actual failover executed (production DR or staging replica)
                 - Clock starts
                 - Team executes runbook without coach assistance
  10:00 - 10:30  Validation: smoke tests, data integrity checks
  10:30 - 11:00  All-clear declared (or escalation if not recovered)
  11:00 - 11:30  Debrief:
                 - Actual RTO vs. target RTO
                 - What worked, what didn't
                 - Runbook gaps found
                 - Action items with owners and due dates
  11:30          Report published to leadership
```

---

## 17. Real-World DR Scenarios

### Scenario 1: AWS Region Outage (e.g., us-east-1)

**What happens:** AWS announces a full `us-east-1` outage affecting EC2, RDS, and EKS.

**Response:**
1. Confirm via AWS Service Health Dashboard (health.aws.amazon.com) — is it us-east-1 or a specific service?
2. Declare P1 incident, activate incident commander
3. Check if warm standby in `eu-west-1` is already running
4. Execute DR runbook: promote Aurora Global secondary → scale EKS → switch Global Accelerator traffic
5. Communicate ETA to stakeholders (don't say "down" — say "switching to DR, ETA 20 minutes")

**Lessons:** Many famous outages (AWS us-east-1 Dec 2021) showed that even AWS Monitoring (console, health dashboard) can be affected. Always have out-of-band communication (phone, Teams/Slack on separate infrastructure).

### Scenario 2: Ransomware Encrypts All EBS Volumes

**What happens:** All EC2 instances in your primary account have their EBS volumes encrypted by ransomware (via a compromised IAM key that had ec2:* permissions).

**Response:**
1. Immediately isolate: revoke the compromised IAM key, isolate affected instances (remove from security groups, stop instances)
2. Check AWS Backup — are backups in a separate account with Vault Lock? If yes — the ransomware couldn't touch them (separate account + Object Lock = untouchable)
3. Restore from last clean AWS Backup recovery point in the isolated DR account
4. Launch in DR VPC (clean environment, not the compromised one)
5. Forensic investigation in parallel: CloudTrail logs to trace the attack path

**Prevention:** Cross-account backup with Vault Lock (COMPLIANCE mode). Separate the backup account from the production account. Even if the production account is fully compromised, the backup account is untouched.

### Scenario 3: Mass Accidental Deletion of S3 Objects

**What happens:** A developer runs `aws s3 rm s3://production-data/ --recursive` targeting a test bucket but accidentally hits production.

**Response:**
1. S3 versioning is enabled → objects are "deleted" but not gone (delete markers added)
2. Remove delete markers to restore objects

```bash
# Bulk restore deleted S3 objects by removing delete markers
aws s3api list-object-versions \
  --bucket production-data \
  --query 'DeleteMarkers[*].{Key:Key,VersionId:VersionId}' \
  --output json > delete-markers.json

# Script to remove all delete markers (restores all objects)
cat delete-markers.json | jq -r '.[] | "\(.Key)\t\(.VersionId)"' | \
while IFS=$'\t' read -r KEY VERSION_ID; do
  aws s3api delete-object \
    --bucket production-data \
    --key "$KEY" \
    --version-id "$VERSION_ID"
  echo "Restored: $KEY"
done
```

**RTO:** Minutes (script runs in parallel). **RPO:** Zero (versioning preserves all versions).

**Prevention:** S3 Object Lock (COMPLIANCE mode) on critical buckets. MFA-delete requirement on versioned buckets. SCP denying `s3:DeleteObject` except from approved IAM roles.

---

## 18. AI & New Trends in DR (2024–2025)

### 1. AWS Resilience Hub (2022, Enhanced 2024)

AWS Resilience Hub is a dedicated service for assessing and improving application resilience. It:
- Automatically discovers your application's architecture (reads CloudFormation/Terraform state)
- Assesses it against your defined RTO/RPO targets
- Identifies gaps ("your EKS cluster has no cross-region failover — current RPO is 4 hours, target is 15 minutes")
- Provides a Resilience Score (0–100)
- Generates an Operational Readiness Review (ORR) report for compliance

```bash
# Create Resilience Hub application
aws resiliencehub create-app \
  --name production-app \
  --description "Core production application" \
  --policy-arn arn:aws:resiliencehub:eu-central-1:123456789012:resiliency-policy/my-policy

# Import Terraform state as input source
aws resiliencehub import-resources-to-draft-app-version \
  --app-arn arn:aws:resiliencehub:eu-central-1:123456789012:app/abc123 \
  --terraform-sources '[{"s3StateFileUrl":"s3://terraform-state/prod/terraform.tfstate"}]'

# Assess resilience
aws resiliencehub start-app-assessment \
  --app-arn arn:aws:resiliencehub:eu-central-1:123456789012:app/abc123 \
  --app-version release \
  --assessment-name "Q4-DR-Assessment"

# Get results
aws resiliencehub describe-app-assessment \
  --assessment-arn arn:aws:resiliencehub:eu-central-1:123456789012:app-assessment/xyz789 \
  --query 'assessment.{Score:resiliencyScore,RTO:meetRtoAndRpoDescription,Findings:componentsWithWarnings}'
```

### 2. Amazon DevOps Guru — Proactive DR Triggering (2024)

DevOps Guru ML detects anomalies that may precede a failure — before the service actually goes down. Examples:
- "Memory utilisation on this RDS instance is increasing 40% faster than normal — may hit OOM in 6 hours"
- "EKS node pool CPU is trending toward saturation during your peak window tomorrow"

Integrate DevOps Guru with EventBridge to auto-trigger pre-emptive actions (take snapshot, scale out, alert on-call) before the incident occurs.

### 3. Parametric DR (2024 Practice)

Instead of fixed DR tiers, organisations are moving to **parametric DR** — the RTO/RPO targets are defined per business transaction type, not per application:
- Payment transaction: RPO = 0, RTO = 30 seconds
- User profile update: RPO = 5 minutes, RTO = 5 minutes
- Reporting query: RPO = 24 hours, RTO = 4 hours

This drives a more granular DR architecture (e.g., DynamoDB Global Tables for payments, S3 backup-restore for reports).

### 4. FinOps-Driven DR Right-Sizing (2024)

With DR costing 15-40% of production infrastructure costs, FinOps teams are applying the same right-sizing principles to DR:
- AWS Compute Optimizer recommends instance sizes for warm standby environments
- Cost Explorer's new "DR cost" tag grouping shows total spend on DR vs. production
- Automated scaling: DR warm standby scales to zero during business hours (saves 60-70% of DR compute costs) and scales up at night for DR testing

### 5. Generative AI for DR Runbook Generation (2024)

Amazon Q can generate DR runbooks from infrastructure definitions:
- Feed it your Terraform code + RTO/RPO requirements
- Q outputs a draft runbook with region-specific commands
- Human review required — but reduces runbook authoring time by 60-70%

### 6. What to Learn Now

- **AWS Resilience Hub** — run it against a personal project to understand the assessment output
- **AWS FIS (Fault Injection Service)** — build a simple chaos experiment
- **Velero** — install on a local kind cluster and practise backup/restore
- **Aurora Global Database** — set up a free-tier lab to understand the failover process
- **AWS Backup + Vault Lock** — understand COMPLIANCE vs. GOVERNANCE mode and when each is legally required

---

## 19. DR Interview Questions & Answers

---

**Q: What is the difference between RTO and RPO? Give a real-world example.**

> "RTO is how long the business can be down — the maximum acceptable outage duration. RPO is how much data can be lost — the age of the last recoverable state.
>
> Example: an e-commerce site processes £10,000 per minute. Business says: 'We can tolerate 15 minutes of downtime before it's critical' → RTO = 15 minutes. And: 'We cannot lose more than 2 minutes of orders' → RPO = 2 minutes.
>
> This maps directly to architecture: 2-minute RPO requires continuous DB replication (Aurora Global, sub-second lag). 15-minute RTO requires warm standby with automated failover — not backup-and-restore which takes hours."

---

**Q: What are the four AWS DR strategies and when would you choose each?**

> "Backup and Restore — cheapest, RTO/RPO in hours. Use for non-critical workloads where hours of downtime is acceptable.
>
> Pilot Light — core DB always replicated, compute scaled to zero. RTO in tens of minutes. Good for applications where downtime costs money but not catastrophically.
>
> Warm Standby — reduced-capacity full stack always running in DR region. RTO in minutes. Use for business-critical apps where an hour of downtime would cause major customer impact.
>
> Multi-Site Active-Active — full capacity in two regions, traffic split between them. RTO near-zero. Use for tier-1 systems where any downtime is unacceptable — payments, authentication, healthcare. Most expensive."

---

**Q: How do you protect a DynamoDB table against accidental deletion?**

> "Multiple layers: Deletion Protection on the table itself (`aws dynamodb update-table --deletion-protection-enabled`), SCP at the org level denying `dynamodb:DeleteTable` except for a restricted IAM role, DynamoDB global tables for multi-region redundancy, and Point-in-Time Recovery (PITR) enabled on the table — which gives 35-day recovery to any second. For the most critical tables, AWS Backup with cross-account copy to an immutable vault. The combination of PITR + cross-account backup + deletion protection means the only way to truly lose a DynamoDB table is a deliberate, authorised multi-step action."

---

**Q: You've been asked to reduce the DR cost for a warm standby environment by 60% without changing the RTO. How?**

> "Warm standby typically has EC2/EKS nodes running 24/7 at reduced capacity — but those nodes are idle 95% of the time. The cost is almost entirely compute.
>
> Solution: schedule the DR compute to scale to zero during business hours and scale up only for:
> 1. Nightly DR drills (automatic scale-up, test, scale-down)
> 2. DR activation (EventBridge rule watching Route 53 health check failure → Lambda → EKS nodegroup scale-up)
>
> The scale-up time for EKS managed node groups is 3-5 minutes. If the business's RTO is 15 minutes, 5 minutes of that can be spent on node provisioning without violating the RTO.
>
> Result: compute cost drops by ~80% (nodes only run for drills and actual DR events). Database replication (Aurora Global) is the remaining cost — no way to avoid that for low RPO. Total DR cost reduction: 60-70%."

---

**Q: What is AWS Resilience Hub and how would you use it in a migration project?**

> "AWS Resilience Hub is a service that assesses your application's resilience against defined RTO/RPO targets. You import your infrastructure definition (CloudFormation stack, Terraform state, or manual resource list), define a resiliency policy (e.g., RTO=15min, RPO=1min, for AZ + region failures), and Resilience Hub runs an assessment that produces a Resilience Score and a list of findings — specific gaps like 'no cross-region replica for this RDS instance.'
>
> In a migration project, I'd use it at two points: first in the Mobilize phase to define baseline resilience targets per application tier, and second in the post-migration optimise phase to validate that the migrated architecture actually meets the targets we designed for. It produces an Operational Readiness Review report, which is useful as evidence for compliance audits."

---

## 20. Quick Reference Cheat Sheet

```
DR Metrics:
  RTO = max acceptable downtime
  RPO = max acceptable data loss
  MTTR = mean time to recover (track over time)
  MTD = maximum tolerable downtime (RTO must be < MTD)
  WRT = work recovery time (validation after restore)
  Total = RTO + WRT ≤ MTD

The Four Strategies (cheapest → fastest):
  Backup & Restore:  RTO hours     RPO hours    Cheapest
  Pilot Light:       RTO 10-30min  RPO minutes  Low cost
  Warm Standby:      RTO 2-15min   RPO seconds  Medium
  Active-Active:     RTO seconds   RPO ~zero    Most expensive

Key Services:
  Compute DR:    AWS Elastic Disaster Recovery (EDR)
  DB DR:         Aurora Global Database, DynamoDB Global Tables, RDS CRR snapshot
  Container DR:  Velero + Argo CD multi-cluster
  Storage DR:    S3 CRR + Object Lock, EFS replication, EBS DLM
  DNS/Traffic:   Route 53 health checks + failover, Global Accelerator
  Backup:        AWS Backup + Vault Lock (COMPLIANCE = truly immutable)
  Networking:    KMS Multi-Region Keys, Secrets Manager replication
  Testing:       AWS FIS (Fault Injection Service), GameDays

RPO by service:
  Aurora Global DB:        < 1 second
  DynamoDB Global Tables:  sub-second
  AWS Elastic DR:          seconds
  S3 CRR:                  seconds–minutes
  RDS Multi-AZ:            zero (sync)
  RDS cross-region replica: minutes
  EFS replication:         < 15 minutes
  AWS Backup (continuous): 5 minutes
  Daily snapshot:          up to 24 hours

RTO by service:
  Route 53 failover:       60s–5min (TTL dependent)
  Global Accelerator:      < 30 seconds
  Aurora Global failover:  < 1 minute
  DynamoDB Global:         < 1 minute (Route 53 re-routes)
  EKS scale-up:            3-8 minutes
  RDS promote replica:     5-15 minutes
  Backup & Restore:        hours

Immutable Backup Hierarchy:
  1. S3 Object Lock (COMPLIANCE) — no one can delete, not even root
  2. AWS Backup Vault Lock (COMPLIANCE) — same, for all backup types
  3. Cross-account backup — separate blast radius from production
  4. Always test restore — untested backup is not a backup
```

---

*References (2023–2025):*
- [AWS Well-Architected — Reliability Pillar (DR Strategies)](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/plan-for-disaster-recovery-dr.html)
- [AWS Elastic Disaster Recovery user guide](https://docs.aws.amazon.com/drs/latest/userguide/what-is-drs.html)
- [Aurora Global Database — failover guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
- [AWS Resilience Hub — Getting started (2023)](https://docs.aws.amazon.com/resilience-hub/latest/userguide/getting-started.html)
- [AWS Backup Vault Lock (2023)](https://docs.aws.amazon.com/aws-backup/latest/devguide/vault-lock.html)
- [AWS FIS — Chaos Engineering on AWS (2023)](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
- [Velero — Kubernetes DR (v1.13, 2024)](https://velero.io/docs/v1.13/)
- [AWS Global Accelerator — failover configuration](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups-traffic-dial.html)
- [Amazon DevOps Guru for proactive resilience (2024)](https://aws.amazon.com/devops-guru/)
