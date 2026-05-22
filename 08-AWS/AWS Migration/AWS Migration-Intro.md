# AWS Cloud Migration — Complete Introduction for DevOps & DevSecOps Engineers

> **Scope of this document:** What cloud migration is, why organizations do it, the frameworks and methodologies that govern it, the four major migration patterns (Lift & Shift, Replatform, Database, Container), DevOps and DevSecOps engineering responsibilities at every phase, Landing Zone design, network architecture, cutover strategies, monitoring, CLI commands, Terraform, and 2024–2025 AI-driven trends.

---

## Table of Contents

1. [What Is AWS Cloud Migration?](#1-what-is-aws-cloud-migration)
2. [Why Organizations Migrate — Business Drivers](#2-why-organizations-migrate--business-drivers)
3. [AWS Cloud Adoption Framework (CAF)](#3-aws-cloud-adoption-framework-caf)
4. [The 7 Rs of Migration](#4-the-7-rs-of-migration)
5. [AWS Migration Acceleration Program (MAP)](#5-aws-migration-acceleration-program-map)
6. [Migration Phases Deep Dive](#6-migration-phases-deep-dive)
7. [AWS Migration Services Catalog](#7-aws-migration-services-catalog)
8. [Landing Zone & Multi-Account Architecture](#8-landing-zone--multi-account-architecture)
9. [Network Architecture for Migration](#9-network-architecture-for-migration)
10. [Pattern 1 — Rehost: On-Premises to AWS (Lift & Shift)](#10-pattern-1--rehost-on-premises-to-aws-lift--shift)
11. [Pattern 2 — Replatform: Lift & Reshape](#11-pattern-2--replatform-lift--reshape)
12. [Pattern 3 — Database Migrations (DMS & SCT)](#12-pattern-3--database-migrations-dms--sct)
13. [Pattern 4 — Container Migrations (EKS & ECS)](#13-pattern-4--container-migrations-eks--ecs)
14. [DevOps Engineering Responsibilities During Migration](#14-devops-engineering-responsibilities-during-migration)
15. [DevSecOps Security Considerations](#15-devsecops-security-considerations)
16. [Cutover Strategies](#16-cutover-strategies)
17. [Post-Migration: Monitoring, Observability & Optimization](#17-post-migration-monitoring-observability--optimization)
18. [Common Migration Pitfalls and How to Avoid Them](#18-common-migration-pitfalls-and-how-to-avoid-them)
19. [AI & New Trends in AWS Migration (2024–2025)](#19-ai--new-trends-in-aws-migration-20242025)
20. [Interview Summary & Key Concepts Cheat Sheet](#20-interview-summary--key-concepts-cheat-sheet)

---

## 1. What Is AWS Cloud Migration?

**AWS Cloud Migration** is the process of moving an organization's digital assets — applications, infrastructure, data, and workloads — from on-premises data centers (or other cloud providers) to Amazon Web Services. It is not a single event but a **structured programme** that can span weeks to years depending on the organisation's size and complexity.

Migration is NOT just "copy your servers to EC2." A comprehensive migration touches:

- **Compute** — physical servers, VMware VMs, bare metal → EC2, EKS, ECS, Lambda
- **Storage** — NAS, SAN, file shares → S3, EFS, FSx, EBS
- **Databases** — Oracle, SQL Server, MySQL, PostgreSQL, MongoDB → RDS, Aurora, DynamoDB, DocumentDB
- **Network** — on-premises firewalls, routers, load balancers → VPC, Transit Gateway, ALB/NLB, WAF
- **Security** — AD, LDAP, PKI, SIEM → IAM, SSO, GuardDuty, Security Hub, KMS
- **Identity** — Active Directory, LDAP → AWS SSO (IAM Identity Center), Cognito, Managed AD
- **Applications** — monolithic apps, .NET, Java EE, SAP → containerized, serverless, or managed equivalents
- **CI/CD Pipelines** — Jenkins on-prem, GitLab self-hosted → CodePipeline, CodeBuild, GitHub Actions, GitLab CI on EKS

### Key Characteristics of a Migration Project

| Characteristic | Description |
|---|---|
| **Duration** | 3 months (small) to 3+ years (enterprise data center exit) |
| **Team** | Cross-functional: DevOps, DBA, Network, Security, App owners, Project Management |
| **Risk** | High — production outages, data loss, compliance violations if poorly planned |
| **Reversibility** | Partial — most migrations have rollback plans for cutover, full reversal is costly |
| **Success metric** | Business outcomes: cost reduction, agility, compliance, not just "moved to cloud" |

---

## 2. Why Organizations Migrate — Business Drivers

Understanding **why** an organisation is migrating is critical for a DevOps engineer. The strategy (7 Rs) and pace directly depend on the driver:

### Primary Drivers

| Driver | Explanation | Migration Urgency |
|---|---|---|
| **Data center lease expiry** | Physical DC contract ends in 12–24 months — hard deadline | High (timeline-driven) |
| **Hardware end-of-life** | Servers reaching end of support — security and vendor risk | High |
| **Cost reduction** | CapEx (hardware purchase) → OpEx (pay-as-you-go) | Medium |
| **Scalability bottleneck** | On-prem cannot scale fast enough for traffic spikes | Medium-High |
| **Compliance & sovereignty** | GDPR, HIPAA, ISO 27001 — easier on managed cloud | Medium |
| **Merger & Acquisition** | Consolidating IT estate from an acquired company | High |
| **Developer velocity** | Slow on-prem provisioning (weeks) vs. cloud (minutes) | Medium |
| **Disaster Recovery** | Replacing expensive DR DC with cloud-native DR | Medium |
| **Innovation** | Access to managed AI/ML (SageMaker, Bedrock), analytics (Redshift, Glue) | Low urgency, high value |

### What This Means for a DevOps Engineer

If the driver is a **data center deadline** → expect a rehost-heavy, fast migration with less re-architecture time.
If the driver is **cost** → expect deeper replatforming, right-sizing, and Reserved Instance planning.
If the driver is **compliance** → expect significant time on Landing Zone, IAM, encryption, and audit trail.

---

## 3. AWS Cloud Adoption Framework (CAF)

The **AWS Cloud Adoption Framework (CAF)** is AWS's structured methodology for enterprise cloud migration. It organises migration readiness across **six perspectives** — three business-focused, three technical-focused.

```
AWS Cloud Adoption Framework (CAF) v3.0 (2021)

  Business Perspectives:          Technical Perspectives:
  ┌─────────────────┐             ┌─────────────────┐
  │   Business      │             │   Platform      │
  │ (ROI, outcomes) │             │ (architecture,  │
  └─────────────────┘             │  IaC, services) │
  ┌─────────────────┐             └─────────────────┘
  │    People       │             ┌─────────────────┐
  │ (skills, org    │             │   Security      │
  │  change mgmt)   │             │ (IAM, encrypt,  │
  └─────────────────┘             │  compliance)    │
  ┌─────────────────┐             └─────────────────┘
  │   Governance    │             ┌─────────────────┐
  │ (risk, finance, │             │  Operations     │
  │  compliance)    │             │ (monitoring,    │
  └─────────────────┘             │  observability) │
                                  └─────────────────┘
```

### The Six CAF Perspectives Explained

**1. Business Perspective**
Ensures cloud investments align to business outcomes. Key activities: cloud business case (TCO analysis), stakeholder alignment, investment prioritisation. Owners: CxO, Finance, PMO.

**2. People Perspective**
Manages organisational change — roles evolving from "server admin" to "cloud engineer". Key activities: skills gap assessment, training programmes, cloud CoE (Centre of Excellence) creation. Owners: HR, L&D, Engineering managers.

**3. Governance Perspective**
Manages risk, compliance, and financial governance. Key activities: portfolio prioritisation, license management, cloud financial management (FinOps). Owners: CIO, Compliance, Legal.

**4. Platform Perspective**
Technical architecture and IaC. Key activities: Landing Zone design, multi-account strategy, reference architectures, IaC standards. **This is where most DevOps time is spent.** Owners: Cloud Architects, DevOps Engineers.

**5. Security Perspective**
End-to-end security posture. Key activities: identity federation, data encryption, network security, threat detection, compliance automation. Owners: CISO, DevSecOps Engineers.

**6. Operations Perspective**
Day-2 operational readiness. Key activities: monitoring frameworks, runbooks, alerting, incident management, backup and DR. Owners: SRE, CloudOps teams.

### CAF Transformation Domains

CAF v3 maps outcomes to four transformation domains:
- **Technology** — modernise legacy IT
- **Process** — automate and optimise business processes  
- **Organisation** — evolve structure and capabilities
- **Product** — accelerate innovation and time-to-market

---

## 4. The 7 Rs of Migration

The **7 Rs** (originally "5 Rs" from Gartner, expanded by AWS) is the taxonomy for classifying what happens to each workload. Every application in the portfolio gets one of these strategies:

```
  Less Change ◄──────────────────────────────────────────► More Change
  Lower Risk                                                 Higher Value
  Faster                                                     Slower

  Retire → Retain → Rehost → Relocate → Replatform → Repurchase → Refactor
```

### R1 — Retire (Decommission)

**What:** Simply turn it off. The application is no longer needed.
**When:** Discovered during assessment that the app has no active users, is superseded by another system, or has a planned sunset.
**DevOps action:** Validate zero active users (CloudWatch equivalent on-prem, log analysis) → get business sign-off → decommission → update CMDB.
**Example:** 15-year-old internal reporting tool replaced by AWS QuickSight.
**Typical portfolio %:** 10–20% of applications in a mature migration.

### R2 — Retain (Keep On-Premises)

**What:** Keep as-is on-premises — deliberately, not by default.
**When:**
- Application cannot move due to regulatory data residency requirements
- Recently upgraded (major investment, no business case to move now)
- Tightly coupled to on-prem hardware (scientific instruments, manufacturing control systems)
- Hybrid architecture where on-prem is intentional (edge computing, latency-sensitive)

**DevOps action:** Document why it's retained, establish review date, ensure hybrid connectivity (VPN/Direct Connect) if it needs to communicate with cloud workloads.
**Example:** Real-time trading system requiring sub-millisecond latency to on-prem exchange connectivity.

### R3 — Rehost (Lift & Shift)

**What:** Move the application to AWS with no changes to the application code or architecture. The server becomes an EC2 instance.
**When:**
- Time pressure (data center deadline)
- Application is a black box (no access to source code)
- Need to get out of the DC quickly, optimise later
- Legacy OS / runtime that cannot be re-architected

**Primary tool:** **AWS Application Migration Service (MGN)**
**DevOps action:** Install MGN agent on source servers → continuous block-level replication to AWS → test launch → cutover (minutes of downtime) → validate → decommission source.
**Example:** 200 VMware VMs replicated to EC2 instances with matching OS/software stack.
**Typical portfolio %:** 50–70% of workloads in a large data-center-exit programme.

**Trade-off:** Cloud bills can be higher than expected — EC2 is not inherently cheaper than on-prem if you don't right-size. Optimisation is a post-migration activity.

### R4 — Relocate (Hypervisor Lift)

**What:** Move VMware workloads to VMware Cloud on AWS (VMC on AWS) — same hypervisor, same tools, now running on AWS hardware.
**When:**
- Organisation is deeply invested in VMware operations and tooling
- Want cloud connectivity without re-training staff on AWS
- Interim step before full cloud-native migration

**DevOps action:** Works at the vSphere/vCenter level — not EC2. Very minimal AWS-native tooling needed initially.
**Example:** Enterprise with 1,000 VMware VMs needing DC exit in 6 months, no time to re-architect.
**Note:** VMC on AWS pricing is significantly higher than native EC2 — it's a bridge strategy, not an end state.

### R5 — Replatform (Lift & Reshape)

**What:** Move to AWS with minor optimisations — not a full re-architecture, but taking advantage of managed services.
**When:**
- Self-managed MySQL → Amazon RDS/Aurora (same MySQL, managed by AWS)
- Tomcat on EC2 → AWS Elastic Beanstalk
- Self-managed Redis → Amazon ElastiCache
- Monolith on VMs → containers on ECS/EKS (same app, containerised)

**DevOps action:** Modify connection strings, configuration files, deployment scripts. Test against managed service. Validate performance and compatibility.
**Example:** Application running on EC2 + self-managed PostgreSQL → EC2 + Aurora PostgreSQL. Same application code, no changes, but DB is now managed — automated backups, Multi-AZ, patching handled by AWS.
**Typical portfolio %:** 20–30% of workloads.

### R6 — Repurchase (Drop & Shop)

**What:** Replace the application with a SaaS alternative.
**When:**
- Custom CRM → Salesforce
- Self-managed email → Microsoft 365 / Google Workspace
- On-prem HR system → Workday
- Self-hosted Jira → Jira Cloud

**DevOps action:** Data migration (export/import), SSO integration (SAML/OIDC via IAM Identity Center), API integration with remaining on-prem or cloud systems.
**Example:** Replacing a self-managed Confluence Data Center with Confluence Cloud — Atlassian manages the infrastructure entirely.

### R7 — Refactor / Re-architect

**What:** Rebuild or significantly redesign the application to be cloud-native. Maximum effort, maximum value.
**When:**
- Application has significant scalability or reliability problems
- Business needs features (e.g., real-time analytics, ML) that require cloud-native services
- Monolith breaking into microservices
- Moving to serverless (Lambda, API Gateway, DynamoDB)

**DevOps action:** This is a full software engineering project — new architecture, new IaC, new CI/CD pipelines, new observability stack, possibly new programming models (event-driven, serverless).
**Example:** Monolithic Java EE application on 40 VMs → 15 microservices on EKS with Kafka, Aurora, Elasticache, and API Gateway.
**Typical portfolio %:** 5–15% of workloads (high effort, reserved for highest-value applications).

### The Portfolio Assessment — Applying the 7 Rs

In practice, a migration starts with a **portfolio assessment** that assigns each application one of the 7 Rs:

```bash
# Application Portfolio Analysis (conceptual)
Total portfolio: 300 applications
  Retire:      45 apps  (15%) → decommission, save licensing cost
  Retain:      30 apps  (10%) → stay on-prem for now
  Rehost:     165 apps  (55%) → lift & shift via MGN
  Relocate:    15 apps   (5%) → VMware Cloud on AWS
  Replatform:  30 apps  (10%) → managed services adoption
  Repurchase:   9 apps   (3%) → SaaS replacements
  Refactor:     6 apps   (2%) → full cloud-native rebuild
```

---

## 5. AWS Migration Acceleration Program (MAP)

The **AWS Migration Acceleration Program (MAP)** is AWS's funded framework that provides financial support, tooling, and methodology for enterprise migrations. It structures the migration into three phases:

```
MAP Phases:
  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────────┐
  │   ASSESS    │───▶│  MOBILIZE   │───▶│   MIGRATE & MODERNIZE    │
  │  (2-4 wks)  │    │ (2-6 months)│    │    (months to years)     │
  └─────────────┘    └─────────────┘    └──────────────────────────┘
  Discover &         Foundation &        Wave execution &
  size the work      readiness           continuous delivery
```

### Phase 1 — Assess

**Goal:** Understand the current state, build the business case, and identify risks.

Key activities:
- **Migration Readiness Assessment (MRA)** — workshops covering all CAF perspectives, produces a readiness score and remediation backlog
- **AWS Migration Evaluator** — agentless collector installed in the environment captures actual utilisation data (CPU, RAM, storage) over 2-4 weeks → generates right-sized EC2 recommendations and TCO comparison
- **Application discovery** — AWS Application Discovery Service (ADS) with agents or agentless (vCenter integration) → populates AWS Migration Hub with server inventory, dependencies, and utilisation data
- **Business case creation** — 3-year TCO model comparing on-prem CAPEX vs. AWS OPEX

**DevOps engineer output from Assess phase:**
- Dependency mapping (which servers talk to which — critical for grouping into migration waves)
- Performance baseline (actual CPU/RAM utilisation for right-sizing EC2 instances)
- Risk register (unsupported OS versions, licensing issues, hard-coded IP addresses)
- Wave plan (logical grouping of servers for phased migration)

```bash
# Install AWS Application Discovery Agent on Linux servers
# Feeds data into Migration Hub for dependency mapping
curl -o ./aws-discovery-agent.tar.gz \
  https://s3-us-west-2.amazonaws.com/aws-discovery-agent/linux/latest/aws-discovery-agent.tar.gz
tar -xzf aws-discovery-agent.tar.gz
sudo bash install -r us-east-1 -k ACCESS_KEY_ID -s SECRET_ACCESS_KEY

# Start discovery
sudo systemctl start aws-discovery-daemon

# Check discovery service in Migration Hub
aws discovery describe-agents \
  --query 'agentsInfo[*].{AgentId:agentId,Status:agentState,Hostname:hostName}'
```

### Phase 2 — Mobilize

**Goal:** Build the foundation for migration — Landing Zone, security baseline, migration runbooks, team upskilling.

Key activities:
- **Landing Zone construction** — AWS Control Tower, multi-account structure, networking, security baseline
- **Migration factory setup** — tooling (MGN, DMS), runbooks, wave planning
- **Security baseline** — GuardDuty, Security Hub, CloudTrail, Config, IAM Identity Center enabled across all accounts
- **CI/CD pipeline** — establish IaC pipeline (Terraform + GitLab/GitHub Actions) for all infrastructure
- **Pilot migration** — 2-5 non-critical servers migrated end-to-end to validate the process before full production wave

**DevOps engineer output from Mobilize phase:**
- Terraform modules for Landing Zone (VPC, subnets, TGW, security groups)
- Runbooks for each migration pattern (Rehost runbook, DB migration runbook, container migration runbook)
- Validated MGN process (test run on pilot servers)
- CI/CD pipelines functional and tested

### Phase 3 — Migrate & Modernize

**Goal:** Execute migration in waves, validate, optimise, and decommission source.

Key activities:
- **Wave execution** — migrate batches of applications (a "wave" = group of related servers migrated together)
- **Cutover management** — change freeze, DNS switch, connection string updates
- **Validation** — smoke tests, integration tests, performance baseline comparison
- **Optimise** — right-size instances, purchase Reserved Instances/Savings Plans, enable auto-scaling
- **Decommission** — terminate source servers after validation period (typically 30 days post-cutover)

**DevOps engineer output from Migrate & Modernize phase:**
- Post-migration validation scripts
- Cost optimisation reports (AWS Cost Explorer, Compute Optimizer recommendations)
- Updated runbooks based on lessons learned from each wave
- Monitoring dashboards (CloudWatch, Grafana) for migrated workloads

---

## 6. Migration Phases Deep Dive

### Wave Planning

A **migration wave** is a logical group of servers/applications migrated together in a single cutover event. Waves are designed based on:

1. **Application dependencies** — all servers that must work together go in the same wave
2. **Business criticality** — critical applications migrate in later waves (after process is validated on less critical apps)
3. **Complexity** — simple standalone apps first, complex multi-tier apps later
4. **Business calendar** — avoid cutting over during peak business periods (e.g., no migrations during Black Friday for retail)

```
Wave Plan Example (50-server migration):
  Wave 0 (Pilot):   3 servers — non-critical dev app — validates process
  Wave 1:          10 servers — internal tools (wiki, monitoring) — low risk
  Wave 2:          15 servers — staging environment — moderate risk
  Wave 3:          12 servers — production tier 2 apps — higher risk
  Wave 4:           8 servers — production tier 1 (core business) — highest risk
  Wave 5:           2 servers — most critical, kept on-prem until all others validated
```

### Migration Testing Strategy

Before every production cutover, run a **test launch**:
- MGN provides test instance launch capability
- Validate application functionality in AWS environment
- Run automated smoke tests (HTTP health checks, DB connection tests, integration tests)
- Compare performance metrics (response time, throughput) between source and AWS
- Validate DNS resolution, firewall rules, IAM permissions

```bash
# MGN: launch test instance before cutover
aws mgn start-test --source-server-ids s-1234567890abcdef0

# Monitor test launch progress
aws mgn describe-source-servers \
  --filters '{"sourceServerIDs":["s-1234567890abcdef0"]}' \
  --query 'items[0].{ID:sourceServerID,State:dataReplicationInfo.dataReplicationState,TestState:lifeCycle.state}'
```

---

## 7. AWS Migration Services Catalog

| Service | Purpose | Primary Use Case |
|---|---|---|
| **AWS Application Migration Service (MGN)** | Continuous block-level server replication | Rehost — P2V/V2V to EC2 |
| **AWS Database Migration Service (DMS)** | Continuous data replication between DB engines | Database migration (homo/hetero) |
| **AWS Schema Conversion Tool (SCT)** | Converts DB schema + stored procedures | Heterogeneous DB migration |
| **AWS DataSync** | High-speed file/object data transfer | NFS/SMB → EFS/S3, S3→S3 |
| **AWS Snow Family** | Offline bulk data transfer (physical devices) | Petabyte-scale or no internet |
| **AWS Transfer Family** | Managed SFTP/FTPS/FTP/AS2 server | File transfer protocol migration |
| **AWS Application Discovery Service** | Server inventory, dependency mapping | Assessment phase |
| **AWS Migration Hub** | Central dashboard for tracking migration | Portfolio tracking |
| **AWS Migration Evaluator** | TCO modelling and right-sizing | Business case creation |
| **AWS VM Import/Export** | Import VM images (OVA, VHD, VMDK) | One-time lift for specific VMs |
| **AWS Server Migration Service (SMS)** | **Deprecated (2022)** — replaced by MGN | Legacy, do not use |
| **AWS Elastic Disaster Recovery (EDR)** | Continuous replication for DR | DR + can be used for migration |
| **AWS DMS Fleet Advisor** | Automated database discovery + migration planning | DB migration assessment |

### AWS Application Migration Service (MGN) — Deep Dive

MGN is the **primary tool for rehost migrations**. It replaced CloudEndure Migration and AWS SMS in 2021.

**How it works:**
1. Install the **MGN Agent** on source servers (Windows or Linux)
2. Agent performs **continuous block-level replication** to a **Replication Server** in AWS (an EC2 instance in your AWS account)
3. Data is encrypted in transit (TLS) and at rest (EBS encryption)
4. **Staging area** maintains a synced copy of the source disk as EBS volumes
5. When ready, **launch a test instance** — validate in AWS without affecting source
6. During the cutover window, **launch the cutover instance** — MGN stops replication and launches the final EC2 instance
7. Minimal downtime cutover (typically 10-60 minutes depending on disk size)

```
MGN Data Flow:
  On-Premises Source         │  AWS Account
  ─────────────────          │  ──────────────────────────────────────
  Server (agent installed)   │  MGN Replication Server (auto-provisioned)
  OS + apps + data ─TLS───▶ │  → Staging EBS volumes (1:1 copy)
                             │         ↓
                             │  Test Instance (for validation)
                             │         ↓
                             │  Cutover Instance (final EC2)
```

```bash
# Install MGN agent on Linux (run on source server)
wget -O ./aws-replication-installer-init.py \
  https://aws-application-migration-service-us-east-1.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py
sudo python3 aws-replication-installer-init.py \
  --region us-east-1 \
  --aws-access-key-id AKIAIOSFODNN7EXAMPLE \
  --aws-secret-access-key wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  --no-prompt

# List source servers in MGN
aws mgn describe-source-servers \
  --query 'items[*].{ID:sourceServerID,Hostname:sourceProperties.identificationHints.hostname,State:dataReplicationInfo.dataReplicationState,Lag:dataReplicationInfo.dataReplicationTimestamps.lastHeartbeat}'

# Initiate test launch
aws mgn start-test \
  --source-server-ids s-1234567890abcdef0

# After validation, initiate cutover
aws mgn start-cutover \
  --source-server-ids s-1234567890abcdef0

# Mark cutover as complete (stops replication, allows decommission)
aws mgn finish-cutover \
  --source-server-ids s-1234567890abcdef0

# Disconnect and terminate replication resources
aws mgn disconnect-from-service \
  --source-server-ids s-1234567890abcdef0
```

---

## 8. Landing Zone & Multi-Account Architecture

A **Landing Zone** is the secure, well-architected, multi-account AWS environment that serves as the foundation for ALL migration workloads. It is built during the Mobilize phase before any application migrations begin.

### Why Multi-Account?

Single-account architecture is an anti-pattern for enterprises:
- **Blast radius** — a security incident, billing mistake, or IAM misconfiguration affects everything
- **Compliance** — different regulations for different workloads require isolation
- **Cost allocation** — separate accounts = clean billing per team/environment/product
- **Service limits** — EC2 vCPU limits, S3 bucket limits, API rate limits are per-account
- **Access control** — least-privilege is far easier across accounts than within one account

### AWS Multi-Account Strategy (Landing Zone Structure)

```
AWS Organizations (Management Account)
├── Root
│   ├── Management Account (billing, org-level SCPs only)
│   │
│   ├── Security OU
│   │   ├── Audit Account (CloudTrail logs, Config findings aggregation)
│   │   └── Log Archive Account (centralized CloudWatch Logs, S3 access logs)
│   │
│   ├── Infrastructure OU
│   │   ├── Network Account (Transit Gateway, Direct Connect, Route 53 Resolver)
│   │   └── Shared Services Account (Active Directory, Nexus, Artifactory, VPN)
│   │
│   ├── Sandbox OU
│   │   └── Individual developer sandbox accounts (with spending limits)
│   │
│   ├── Workloads OU
│   │   ├── Dev OU
│   │   │   ├── AppA-Dev Account
│   │   │   └── AppB-Dev Account
│   │   ├── Test OU
│   │   │   ├── AppA-Test Account
│   │   │   └── AppB-Test Account
│   │   └── Prod OU
│   │       ├── AppA-Prod Account
│   │       └── AppB-Prod Account
│   │
│   └── Migration OU (temporary — during migration project)
│       ├── Migration-Wave1 Account
│       └── Migration-Wave2 Account
```

### AWS Control Tower — Landing Zone Automation

**AWS Control Tower** automates the creation and governance of a multi-account environment. It provisions the above structure via **Account Factory** and enforces **Guardrails** (SCPs + Config rules).

```bash
# Control Tower enables these by default at landing zone setup:
# - AWS Organizations
# - AWS SSO (IAM Identity Center)
# - CloudTrail (org-level trail)
# - AWS Config (aggregator)
# - VPC default rules (no default VPC in new accounts)

# Account Factory — provision new AWS accounts via CLI
aws servicecatalog provision-product \
  --product-name "AWS Control Tower Account Factory" \
  --provisioning-artifact-name "AWS Control Tower Account Factory" \
  --provisioned-product-name "AppA-Production" \
  --provisioning-parameters \
    ParameterKey=AccountName,ParameterValue=AppA-Production \
    ParameterKey=AccountEmail,ParameterValue=aws+appa-prod@company.com \
    ParameterKey=ManagedOrganizationalUnit,ParameterValue="Prod OU (ou-abc1-12345678)" \
    ParameterKey=SSOUserEmail,ParameterValue=admin@company.com \
    ParameterKey=SSOUserFirstName,ParameterValue=Cloud \
    ParameterKey=SSOUserLastName,ParameterValue=Admin

# List guardrails (preventive + detective controls)
aws controltower list-enabled-controls \
  --target-identifier arn:aws:organizations::123456789012:ou/o-abc123/ou-xyz789
```

### Service Control Policies (SCPs) — Guardrails

SCPs are IAM-like policies attached to OUs or accounts in AWS Organizations. They set the **maximum permissions ceiling** — even an account-level Administrator cannot exceed what the SCP allows.

```json
// SCP: Prevent resources being created outside approved regions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "route53:*",
        "budgets:*",
        "waf:*",
        "cloudfront:*",
        "sts:*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "eu-central-1",
            "eu-west-1",
            "us-east-1"
          ]
        }
      }
    }
  ]
}
```

```json
// SCP: Prevent disabling security services
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisablingSecurityServices",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "securityhub:DisableSecurityHub",
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder",
        "config:DeleteDeliveryChannel",
        "access-analyzer:DeleteAnalyzer"
      ],
      "Resource": "*"
    }
  ]
}
```

### Terraform for Landing Zone

```hcl
# Core Landing Zone components via Terraform

# AWS Organizations
resource "aws_organizations_organization" "main" {
  aws_service_access_principals = [
    "cloudtrail.amazonaws.com",
    "config.amazonaws.com",
    "sso.amazonaws.com",
    "controltower.amazonaws.com",
    "securityhub.amazonaws.com",
    "guardduty.amazonaws.com",
    "access-analyzer.amazonaws.com",
    "ram.amazonaws.com"
  ]
  feature_set = "ALL"
}

# Organizational Units
resource "aws_organizations_organizational_unit" "security" {
  name      = "Security"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "workloads_prod" {
  name      = "Prod"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

# SCP — deny leaving organization (protect against account hijacking)
resource "aws_organizations_policy" "deny_leave_org" {
  name        = "DenyLeaveOrganization"
  description = "Prevent any account from leaving the organization"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Deny"
      Action   = "organizations:LeaveOrganization"
      Resource = "*"
    }]
  })
}

resource "aws_organizations_policy_attachment" "deny_leave_root" {
  policy_id = aws_organizations_policy.deny_leave_org.id
  target_id = aws_organizations_organization.main.roots[0].id
}

# Org-level CloudTrail
resource "aws_cloudtrail" "org_trail" {
  name                          = "org-management-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.bucket
  is_multi_region_trail         = true
  is_organization_trail         = true
  enable_log_file_validation    = true
  include_global_service_events = true
  kms_key_id                    = aws_kms_key.cloudtrail.arn

  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }

  tags = {
    Purpose = "OrgAuditTrail"
  }
}
```

---

## 9. Network Architecture for Migration

The network is the backbone of any migration. It must be established and validated in the Mobilize phase before any servers move.

### Core Network Components

```
On-Premises Network              AWS Cloud
────────────────────             ──────────────────────────────────────────
Corp HQ DC (10.0.0.0/8)         Network Account
  │                               ├── Transit Gateway (TGW)
  ├── AWS Direct Connect ─────▶  │   ├── VPC Prod (10.1.0.0/16)
  │   (1-100 Gbps dedicated)     │   │   ├── Private subnets (app tier)
  │                               │   │   ├── Private subnets (DB tier)
  └── Site-to-Site VPN ──────▶  │   │   └── Public subnets (ALB/NLB)
      (IPSec over internet,       │   ├── VPC Dev (10.2.0.0/16)
       backup to Direct Connect)  │   ├── VPC Staging (10.3.0.0/16)
                                  │   └── VPC Shared Services (10.4.0.0/16)
                                  │       (AD, DNS, NTP, Nexus)
                                  │
                                  └── Route 53 Resolver
                                      (resolve on-prem DNS from AWS
                                       and AWS DNS from on-prem)
```

### AWS Direct Connect vs. Site-to-Site VPN

| Feature | Direct Connect | Site-to-Site VPN |
|---|---|---|
| **Bandwidth** | 1 Gbps to 100 Gbps | Up to 1.25 Gbps |
| **Latency** | Consistent, low latency | Variable (internet) |
| **Security** | Private, not over internet | IPSec encrypted over internet |
| **Cost** | Higher (port fee + data transfer) | Lower (per-hour + data transfer) |
| **Setup time** | Weeks to months (physical circuit) | Minutes to hours |
| **Use for migration** | Primary channel for bulk data | Backup or initial connectivity |
| **Reliability** | Very high (SLA) | Lower (internet-dependent) |

**During migration:** Use Direct Connect for bulk server replication (MGN, DMS). VPN as backup path.

```bash
# Create Site-to-Site VPN connection (while Direct Connect is being provisioned)
# Customer Gateway (on-prem VPN device)
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.10 \  # On-prem router public IP
  --bgp-asn 65000

# Virtual Private Gateway
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512

# Attach VGW to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-12345678 \
  --vpc-id vpc-12345678

# Create VPN connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-12345678 \
  --vpn-gateway-id vgw-12345678 \
  --options '{"StaticRoutesOnly":false,"TunnelOptions":[
    {"TunnelInsideCidr":"169.254.10.0/30"},
    {"TunnelInsideCidr":"169.254.11.0/30"}
  ]}'
```

### Transit Gateway — Hub-and-Spoke Networking

AWS Transit Gateway (TGW) is the **network hub** that connects all VPCs and on-premises networks. Without TGW, VPC peering becomes an n² mess as accounts scale.

```
Without TGW (VPC Peering mesh — nightmare at scale):
  VPC-A ↔ VPC-B ↔ VPC-C ↔ VPC-D → 6 peering connections for 4 VPCs
  n*(n-1)/2 connections: at 10 VPCs = 45 peering connections

With TGW (hub-and-spoke):
  VPC-A ─┐
  VPC-B ─┤
  VPC-C ─┼──▶ Transit Gateway ◀── Direct Connect / VPN (on-prem)
  VPC-D ─┤
  VPC-E ─┘
  → 5 TGW attachments for 5 VPCs, regardless of how many VPCs
```

```hcl
# Terraform: Transit Gateway for Landing Zone
resource "aws_ec2_transit_gateway" "main" {
  description                     = "Central network hub for cloud migration"
  amazon_side_asn                 = 64512
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "enable"
  dns_support                     = "enable"
  vpn_ecmp_support                = "enable"

  tags = {
    Name        = "landing-zone-tgw"
    Environment = "shared"
  }
}

# Attach production VPC to TGW
resource "aws_ec2_transit_gateway_vpc_attachment" "prod" {
  subnet_ids         = aws_subnet.prod_private[*].id
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.prod.id

  dns_support                                     = "enable"
  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  tags = {
    Name = "tgw-attach-prod"
  }
}

# TGW route table for production traffic
resource "aws_ec2_transit_gateway_route_table" "prod" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name = "tgw-rt-prod"
  }
}
```

### Route 53 Resolver for Hybrid DNS

During migration, applications in AWS need to resolve on-premises hostnames (e.g., `db01.corp.internal`) and vice versa.

```hcl
# Inbound resolver endpoint — allows on-prem to query AWS DNS
resource "aws_route53_resolver_endpoint" "inbound" {
  name      = "inbound-resolver"
  direction = "INBOUND"

  security_group_ids = [aws_security_group.resolver.id]

  ip_address {
    subnet_id = aws_subnet.private_a.id
  }

  ip_address {
    subnet_id = aws_subnet.private_b.id
  }
}

# Outbound resolver endpoint — allows AWS to query on-prem DNS
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "outbound-resolver"
  direction = "OUTBOUND"

  security_group_ids = [aws_security_group.resolver.id]

  ip_address {
    subnet_id = aws_subnet.private_a.id
  }

  ip_address {
    subnet_id = aws_subnet.private_b.id
  }
}

# Forward rule: corp.internal queries go to on-prem DNS
resource "aws_route53_resolver_rule" "corp_internal" {
  domain_name          = "corp.internal"
  name                 = "forward-corp-internal"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip {
    ip   = "10.0.0.5"   # On-prem DNS server 1
    port = 53
  }

  target_ip {
    ip   = "10.0.0.6"   # On-prem DNS server 2
    port = 53
  }
}
```

---

## 10. Pattern 1 — Rehost: On-Premises to AWS (Lift & Shift)

This is the most common migration pattern. The goal is speed and risk reduction — get off on-premises hardware quickly, optimise later.

### Full Rehost Workflow

```
Phase 1: Pre-Migration Preparation
  ├── Install MGN agents on source servers
  ├── Configure MGN replication settings (subnet, security groups, instance type)
  ├── Establish network connectivity (Direct Connect or VPN)
  └── Create target EC2 launch template (instance type, EBS config, IAM profile)

Phase 2: Replication (can take hours to days depending on disk size)
  ├── Initial sync: full block-level copy of source disk
  ├── Continuous sync: incremental changes (typically 1-5 Gbps bandwidth)
  └── Monitor lag: should be < 60 seconds before cutover

Phase 3: Test Launch (no impact on source)
  ├── Launch test EC2 instance from replicated data
  ├── Run smoke tests, integration tests
  ├── Validate application functionality, DB connectivity
  ├── Performance comparison: latency, throughput
  └── Terminate test instance

Phase 4: Cutover Window
  ├── Schedule change window (low traffic period)
  ├── Notify stakeholders
  ├── Final replication sync (let lag drop to < 10 seconds)
  ├── Stop application on source server (maintenance mode)
  ├── Initiate MGN cutover → launches final EC2 instance
  ├── Update DNS records (A records / CNAME) → new EC2 IP
  ├── Validate application on AWS
  └── If success: mark cutover complete

Phase 5: Post-Cutover (30-day validation period)
  ├── Monitor CloudWatch metrics (CPU, memory, network, errors)
  ├── Keep source server running but idle (rollback option)
  ├── After 30 days with no issues: decommission source
  └── Right-size EC2 instance using Compute Optimizer recommendations
```

### Right-Sizing with AWS Compute Optimizer

After migration, right-sizing is critical. AWS Compute Optimizer analyses CloudWatch metrics and recommends optimal instance types:

```bash
# Get Compute Optimizer recommendations for EC2 instances
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'instanceRecommendations[*].{
    InstanceARN:instanceArn,
    Current:currentInstanceType,
    Recommended:recommendationOptions[0].instanceType,
    SavingsPercent:recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table

# Get Lambda function recommendations
aws compute-optimizer get-lambda-function-recommendations \
  --query 'lambdaFunctionRecommendations[*].{
    Function:functionArn,
    Finding:finding,
    CurrentMemory:memorySizeRecommendationOptions[0].memorySize
  }'
```

### Handling Common Rehost Challenges

**Challenge 1: Hard-coded IP addresses in application config**
```bash
# Find hard-coded IPs in config files after migration
grep -rn "10\.\|192\.168\.\|172\." /etc/app/config/ | grep -v "#"
# Solution: Replace IPs with DNS names before cutover, use Route 53 private hosted zones
```

**Challenge 2: Windows Licensing**
- Windows Server licenses on AWS EC2 are included in the hourly rate (license included)
- If you bring your own Windows license (BYOL), you need **Dedicated Hosts**
- SQL Server BYOL also requires Dedicated Hosts

```bash
# Launch on Dedicated Host for BYOL licensing
aws ec2 allocate-hosts \
  --instance-type m5.xlarge \
  --quantity 1 \
  --availability-zone us-east-1a \
  --auto-placement on \
  --host-recovery on
```

**Challenge 3: Applications requiring specific kernel modules or drivers**
```bash
# Check kernel version compatibility after launch
uname -r
# Install required kernel modules
sudo yum install kernel-modules-extra -y
# For GPU workloads: install NVIDIA drivers
sudo dnf install -y nvidia-driver
```

---

## 11. Pattern 2 — Replatform: Lift & Reshape

Replatforming applies when a small change delivers significant operational benefit. The most common examples:

### Self-Managed DB → Amazon RDS/Aurora

```
Before: EC2 instance running MySQL 8.0, manually managed
  - Manual backups (cron + mysqldump)
  - Manual failover (DNS change + slave promotion)
  - Manual patching (downtime window each month)
  - No automated performance insights

After: Amazon RDS MySQL 8.0 / Aurora MySQL
  - Automated daily backups + PITR
  - Multi-AZ automatic failover (60 seconds)
  - Automated minor version patching
  - Performance Insights, Enhanced Monitoring built-in
```

**Migration path:** Use DMS for the DB move (covered in Section 12).

### Application Server → Elastic Beanstalk

```bash
# Package application for Elastic Beanstalk
# Java WAR file
eb init my-application --platform java-8 --region us-east-1
eb create production-env \
  --instance-type t3.medium \
  --min-instances 2 \
  --max-instances 10 \
  --database \
  --database.engine mysql \
  --database.instance db.t3.medium

# Deploy new version
eb deploy production-env
```

### Containerising a Monolith for ECS/EKS

The first step of replatforming a monolith to containers is **containerisation** — not yet decomposing into microservices.

```dockerfile
# Dockerfile: Containerise existing Java WAR application
FROM amazoncorretto:17-alpine

# Install dependencies
RUN apk add --no-cache curl

# Copy application
COPY target/myapp.war /opt/tomcat/webapps/ROOT.war

# Configuration via environment variables (12-factor)
ENV DB_HOST=${DB_HOST}
ENV DB_PORT=${DB_PORT}
ENV DB_NAME=${DB_NAME}

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["java", "-jar", "/opt/tomcat/webapps/ROOT.war"]
```

```bash
# Build and push to ECR
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1

# Create ECR repository
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS

# Login to ECR
aws ecr get-login-password --region $REGION | \
  docker login --username AWS \
  --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

# Build, tag, push
docker build -t myapp:v1.0.0 .
docker tag myapp:v1.0.0 $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/myapp:v1.0.0
docker push $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/myapp:v1.0.0
```

---

## 12. Pattern 3 — Database Migrations (DMS & SCT)

Database migrations are the highest-risk component of any cloud migration. A failed DB cutover means data loss or corruption — the most severe incident type.

### Two Categories of DB Migration

**Homogeneous Migration:** Same engine, same version (or similar).
```
MySQL → MySQL (RDS/Aurora)
PostgreSQL → PostgreSQL (RDS/Aurora)
SQL Server → SQL Server (RDS)
```
Uses: Native DB tools (mysqldump, pg_dump, backup/restore). DMS can also be used for continuous replication.

**Heterogeneous Migration:** Different engine — requires schema conversion.
```
Oracle → Aurora PostgreSQL
SQL Server → Aurora MySQL
MySQL → DynamoDB
Oracle → Aurora MySQL
```
Uses: AWS Schema Conversion Tool (SCT) for schema + stored procedure conversion, then DMS for data migration.

### AWS Schema Conversion Tool (SCT)

SCT is a **desktop application** (runs on your workstation) that:
1. Connects to the source database
2. Analyses the schema, stored procedures, functions, views, and triggers
3. Converts what it can automatically
4. Flags what requires manual conversion (complex stored procedures, proprietary SQL features)
5. Generates a migration assessment report showing conversion complexity

```
SCT Assessment Report (Oracle → Aurora PostgreSQL):
  ┌────────────────────────────────────────────────────────┐
  │ Object Type     │ Total │ Auto-converted │ Manual fix  │
  │─────────────────│───────│────────────────│─────────────│
  │ Tables          │  187  │     187 (100%) │      0      │
  │ Views           │   43  │      40 ( 93%) │      3      │
  │ Stored Procs    │   78  │      45 ( 58%) │     33      │
  │ Functions       │   21  │      18 ( 86%) │      3      │
  │ Triggers        │   34  │      28 ( 82%) │      6      │
  │ Indexes         │  412  │     412 (100%) │      0      │
  └────────────────────────────────────────────────────────┘
  Estimated effort: 45 person-days for manual conversions
```

### AWS Database Migration Service (DMS) — Architecture

DMS works as a **continuous replication engine** — it keeps source and target in sync while migration is in progress, enabling near-zero-downtime database cutovers.

```
DMS Architecture:
  ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
  │  Source DB      │────────▶│  DMS Replication │────────▶│  Target DB      │
  │  (on-prem or   │ CDC via  │  Instance        │  writes  │  (RDS/Aurora/  │
  │   existing RDS) │ binlog/  │  (EC2 in your   │          │   DynamoDB)     │
  └─────────────────┘  redo   │   VPC)           │          └─────────────────┘
                       logs   └─────────────────┘
```

**DMS Migration types:**
1. **Full load** — migrate all existing data once (initial bulk load)
2. **Full load + CDC** — initial load + capture ongoing changes → near-zero-downtime migration
3. **CDC only** — source already has target in sync, only replicate changes from a specific point

```bash
# Create DMS replication instance
aws dms create-replication-instance \
  --replication-instance-identifier prod-dms-instance \
  --replication-instance-class dms.r5.xlarge \
  --allocated-storage 100 \
  --vpc-security-group-ids sg-12345678 \
  --replication-subnet-group-identifier my-dms-subnet-group \
  --multi-az \
  --publicly-accessible false \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123

# Create source endpoint (Oracle on-prem)
aws dms create-endpoint \
  --endpoint-identifier oracle-source \
  --endpoint-type source \
  --engine-name oracle \
  --username dms_user \
  --password "${DB_PASSWORD}" \
  --server-name oracle-db.corp.internal \
  --port 1521 \
  --database-name PRODDB \
  --extra-connection-attributes "useLogminerReader=N;useBfile=Y"

# Create target endpoint (Aurora PostgreSQL)
aws dms create-endpoint \
  --endpoint-identifier aurora-pg-target \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --username admin \
  --password "${AURORA_PASSWORD}" \
  --server-name myaurora.cluster-xxxx.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --database-name appdb

# Test source endpoint connectivity
aws dms test-connection \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:prod-dms-instance \
  --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:oracle-source

# Create replication task (full load + CDC)
aws dms create-replication-task \
  --replication-task-identifier oracle-to-aurora-full-cdc \
  --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:oracle-source \
  --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:aurora-pg-target \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:prod-dms-instance \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [{
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all",
      "object-locator": {"schema-name": "APP", "table-name": "%"},
      "rule-action": "include"
    }]
  }' \
  --replication-task-settings '{
    "TargetMetadata": {"TargetSchema": "public", "SupportLobs": true},
    "FullLoadSettings": {"TargetTablePrepMode": "DO_NOTHING"},
    "Logging": {"EnableLogging": true}
  }'

# Start the replication task
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:us-east-1:123456789012:task:oracle-to-aurora-full-cdc \
  --start-replication-task-type start-replication

# Monitor replication progress
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:... \
  --query 'ReplicationTasks[0].{Status:Status,FullLoadProgress:ReplicationTaskStats.FullLoadProgressPercent,TablesLoaded:ReplicationTaskStats.TablesLoaded,CDCLatency:ReplicationTaskStats.CDCLatencySource}'
```

### Database Cutover Procedure

```
DB Cutover Runbook (near-zero-downtime):

T-7 days:    DMS full load completes, CDC running, lag < 5 seconds
T-1 day:     Validate data consistency (row counts, checksum spot checks)
T-0 (cutover window, e.g., 2AM Sunday):
  1. Put application in maintenance mode (show maintenance page)
  2. Wait for DMS CDC lag to reach 0 seconds
  3. Stop DMS replication task
  4. Run final data validation (row counts, random sample comparison)
  5. Update application connection strings (Secrets Manager or SSM Parameter Store)
  6. Restart application services
  7. Run smoke tests against Aurora
  8. If PASS: maintenance mode off, done (downtime ≈ 10-30 minutes)
  9. If FAIL: revert connection strings, restart on Oracle, investigate
```

---

## 13. Pattern 4 — Container Migrations (EKS & ECS)

Container migrations involve either:
1. **Containerising existing apps** (currently running as VMs) and deploying to EKS/ECS
2. **Migrating an existing Kubernetes cluster** (on-prem kubeadm/Rancher) to Amazon EKS
3. **Migrating Docker Swarm** workloads to ECS or EKS

### On-Premises Kubernetes → Amazon EKS Migration

```
Migration Path:
  On-Prem K8s Cluster          Amazon EKS
  ─────────────────────        ─────────────────────────────────
  kubeadm cluster         →    EKS managed control plane
  Self-managed nodes      →    Managed node groups / Fargate
  Self-managed etcd       →    AWS-managed etcd (Multi-AZ, encrypted)
  Calico CNI              →    AWS VPC CNI
  MetalLB LoadBalancer    →    AWS Load Balancer Controller
  NFS storage             →    Amazon EFS + EFS CSI Driver
  Secrets in etcd         →    AWS Secrets Manager + Secrets Store CSI Driver
  Prometheus/Grafana      →    Amazon Managed Grafana + Amazon Managed Prometheus
  Self-hosted registry    →    Amazon ECR
```

**Step 1: Export workloads from source cluster**

```bash
# Export all Kubernetes manifests from source cluster
# Use Velero for full cluster backup (includes PV data)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-node-agent

# Create full cluster backup
velero backup create pre-migration-backup \
  --include-namespaces '*' \
  --include-resources '*' \
  --default-volumes-to-fs-backup

# Export individual namespace manifests (alternative)
kubectl get all,configmap,secret,pvc,ingress -n production -o yaml > production-backup.yaml
```

**Step 2: Create Amazon EKS Cluster**

```bash
# Create EKS cluster with eksctl
cat > cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-eks-cluster
  region: us-east-1
  version: "1.30"
  tags:
    Environment: production
    MigratedFrom: onprem-k8s

vpc:
  id: vpc-12345678
  subnets:
    private:
      us-east-1a: {id: subnet-aaa}
      us-east-1b: {id: subnet-bbb}
      us-east-1c: {id: subnet-ccc}

privateCluster:
  enabled: true  # Private API server endpoint

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
  - name: amazon-cloudwatch-observability
    version: latest

managedNodeGroups:
  - name: general-workers
    instanceType: m5.xlarge
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
    privateNetworking: true
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
    labels:
      workload-type: general
    tags:
      Environment: production
EOF

eksctl create cluster -f cluster.yaml
```

**Step 3: Install Essential Add-ons**

```bash
# AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=prod-eks-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/AWSLoadBalancerControllerRole

# Cluster Autoscaler (or Karpenter — preferred in 2024)
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version 0.37.0 \
  --namespace karpenter \
  --create-namespace \
  --set settings.clusterName=prod-eks-cluster \
  --set settings.interruptionQueue=prod-eks-cluster

# External Secrets Operator (for AWS Secrets Manager integration)
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace

# EFS CSI Driver (for persistent storage migration from NFS)
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system

# AWS Secrets Manager → Kubernetes Secrets (Secrets Store CSI Driver)
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true
```

**Step 4: Migrate Persistent Storage (NFS → EFS)**

```bash
# Create EFS filesystem
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123 \
  --tags Key=Name,Value=eks-shared-storage Key=Environment,Value=production

# Mount targets in each AZ
for SUBNET in subnet-aaa subnet-bbb subnet-ccc; do
  aws efs create-mount-target \
    --file-system-id fs-12345678 \
    --subnet-id $SUBNET \
    --security-groups sg-efs-12345678
done

# Use DataSync to copy data from on-prem NFS to EFS
aws datasync create-location-nfs \
  --server-hostname nfs-server.corp.internal \
  --subdirectory /exports/appdata \
  --on-prem-config AgentArns=arn:aws:datasync:us-east-1:123456789012:agent/agent-abc123

aws datasync create-location-efs \
  --efs-filesystem-arn arn:aws:elasticfilesystem:us-east-1:123456789012:file-system/fs-12345678 \
  --ec2-config SecurityGroupArns=arn:aws:ec2:us-east-1:123456789012:security-group/sg-efs,SubnetArn=arn:aws:ec2:us-east-1:123456789012:subnet/subnet-aaa

# Create and start DataSync task
aws datasync create-task \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-nfs \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-efs \
  --options "VerifyMode=ONLY_FILES_TRANSFERRED,PreserveDeletedFiles=REMOVE,TransferMode=CHANGED"

aws datasync start-task-execution \
  --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-12345678
```

**Step 5: IRSA — Pod Identity for AWS API Access**

Pods in EKS need IAM permissions to access S3, DynamoDB, Secrets Manager, etc. IRSA (IAM Roles for Service Accounts) replaces hard-coded credentials:

```bash
# Enable OIDC provider for EKS cluster
eksctl utils associate-iam-oidc-provider \
  --cluster prod-eks-cluster \
  --region us-east-1 \
  --approve

# Create IAM role for service account
eksctl create iamserviceaccount \
  --cluster prod-eks-cluster \
  --namespace production \
  --name app-service-account \
  --role-name EKSAppRole \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --attach-policy-arn arn:aws:iam::123456789012:policy/AppCustomPolicy \
  --approve \
  --override-existing-serviceaccounts
```

---

## 14. DevOps Engineering Responsibilities During Migration

As a DevOps engineer in a migration project, your responsibilities span all three MAP phases:

### Phase 1 — Assess

```
DevOps Responsibilities:
✓ Run AWS Application Discovery Service agents for dependency mapping
✓ Analyse utilisation data → right-sizing recommendations
✓ Review existing CI/CD pipelines (what needs to change for AWS?)
✓ Document all application configuration parameters (env vars, connection strings)
✓ Identify all external integrations (third-party APIs, on-prem services)
✓ Security assessment: open ports, service accounts, secret management gaps
✓ Contribute to wave planning: group applications by dependency clusters
```

### Phase 2 — Mobilize (Foundation Building)

This is where most DevOps hands-on work happens:

```
Infrastructure as Code:
✓ Write Terraform modules for:
  - VPC, subnets, route tables, security groups
  - Transit Gateway and attachments
  - EKS cluster and node groups
  - RDS/Aurora clusters
  - ALB/NLB load balancers
  - S3 buckets with lifecycle policies
  - KMS keys and key policies
  - IAM roles and policies

CI/CD Pipeline Setup:
✓ GitLab/GitHub repository structure for IaC
✓ Terraform pipeline (validate → plan → apply on merge)
✓ Application build pipeline (test → build → ECR push → deploy)
✓ GitOps: Argo CD or Flux for Kubernetes deployments

Operational Readiness:
✓ CloudWatch dashboards and alarms for all migrated workloads
✓ Centralised logging (CloudWatch Logs → S3 → optional Elasticsearch/OpenSearch)
✓ Runbooks for common operations (scale up, DB failover, certificate renewal)
✓ DR runbooks and tested backup/restore procedures
```

### Phase 3 — Migrate & Modernize

```
Per-Wave Execution:
✓ Configure MGN replication for wave servers
✓ Monitor replication lag → ensure < 60 seconds before cutover
✓ Execute test launches and validate
✓ Coordinate cutover window with stakeholders
✓ Update DNS, load balancer targets, connection strings
✓ Monitor CloudWatch for 48h post-cutover
✓ Update runbooks with lessons learned

Post-Wave:
✓ Right-size EC2 instances using Compute Optimizer
✓ Purchase Savings Plans/Reserved Instances for stable workloads
✓ Tag resources for cost allocation
✓ Decommission source servers after 30-day validation
```

### IaC Best Practices for Migration Projects

```hcl
# Module structure for migration IaC repo
# ───────────────────────────────────────
# modules/
#   ├── networking/       VPC, subnets, TGW, security groups
#   ├── compute/          EC2, launch templates, ASGs
#   ├── database/         RDS, Aurora, parameter groups
#   ├── eks/              EKS cluster, node groups, add-ons
#   ├── security/         IAM roles, KMS, GuardDuty, Config
#   └── monitoring/       CloudWatch, alarms, dashboards
#
# environments/
#   ├── dev/
#   │   ├── main.tf       (calls modules with dev-specific vars)
#   │   └── terraform.tfvars
#   ├── staging/
#   └── prod/

# Example: Terraform remote state for migration project
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "migration/prod/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
    dynamodb_table = "terraform-state-lock"
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# State locking table
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.terraform.arn
  }
}
```

### CI/CD Pipeline for Infrastructure (GitLab CI Example)

```yaml
# .gitlab-ci.yml — Terraform pipeline for migration infrastructure

stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "environments/prod"
  AWS_DEFAULT_REGION: "us-east-1"

.terraform_base:
  image: hashicorp/terraform:1.8.0
  before_script:
    - cd $TF_ROOT
    - terraform init -backend-config="bucket=${TF_STATE_BUCKET}"
    - terraform workspace select ${TF_WORKSPACE} || terraform workspace new ${TF_WORKSPACE}

validate:
  extends: .terraform_base
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check -recursive
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

plan:
  extends: .terraform_base
  stage: plan
  script:
    - terraform plan -out=plan.tfplan
    - terraform show -json plan.tfplan > plan.json
  artifacts:
    paths:
      - $TF_ROOT/plan.tfplan
    expire_in: 7 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

apply:
  extends: .terraform_base
  stage: apply
  script:
    - terraform apply plan.tfplan
  dependencies:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual  # Manual approval for production apply
  environment:
    name: production
```

---

## 15. DevSecOps Security Considerations

Security must be embedded at **every phase** of migration — not bolted on at the end.

### Security Baseline — Non-Negotiables Before First Wave

Every account provisioned by the Landing Zone must have these enabled:

```bash
# 1. GuardDuty — threat detection (AI/ML-based)
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --data-sources '{"S3Logs":{"Enable":true},"EKSAuditLogs":{"Enable":true},"EBSVolumes":{"Enable":true},"RDSLoginEvents":{"Enable":true},"LambdaNetworkLogs":{"Enable":true}}'

# 2. Security Hub — aggregated security findings
aws securityhub enable-security-hub \
  --enable-default-standards  # Enables CIS Foundations, AWS Foundational, PCI-DSS

# 3. AWS Config — configuration compliance
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

# 4. CloudTrail — API audit log
aws cloudtrail create-trail \
  --name org-cloudtrail \
  --s3-bucket-name company-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-cloudtrail

# 5. IAM Access Analyzer — detect unintended public resource exposure
aws accessanalyzer create-analyzer \
  --analyzer-name landing-zone-analyzer \
  --type ORGANIZATION
```

### Secrets Management — No Hard-Coded Credentials

One of the biggest security debts in migrated applications is hard-coded database passwords, API keys, and service account credentials. The migration is an opportunity to remediate:

```bash
# Store DB credentials in Secrets Manager
aws secretsmanager create-secret \
  --name "prod/myapp/database" \
  --description "Production database credentials" \
  --secret-string '{
    "username": "appuser",
    "password": "verysecurepassword",
    "host": "myaurora.cluster-xxxx.us-east-1.rds.amazonaws.com",
    "port": "5432",
    "dbname": "appdb"
  }' \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/mrk-abc123

# Enable automatic rotation (30-day rotation for Aurora)
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/database \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRotationLambda \
  --rotation-rules AutomaticallyAfterDays=30

# Application reads from Secrets Manager (no hard-coded credentials)
# Python example:
# import boto3, json
# client = boto3.client('secretsmanager')
# secret = json.loads(client.get_secret_value(SecretId='prod/myapp/database')['SecretString'])
```

### Network Security During Migration

```hcl
# Security Group — principle of least privilege
resource "aws_security_group" "app_tier" {
  name        = "app-tier-sg"
  description = "Application tier — allow only from ALB"
  vpc_id      = aws_vpc.prod.id

  # Only allow traffic from ALB security group (not open internet)
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "HTTPS from ALB only"
  }

  # Egress: only to DB tier and specific AWS endpoints
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.db_tier.id]
    description     = "PostgreSQL to DB tier"
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS to AWS APIs (via VPC endpoints preferred)"
  }

  # NO egress "allow all" — this is a common migration security gap
  tags = {
    Name = "app-tier-sg"
  }
}

# VPC Endpoints — keep AWS API traffic off the internet
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.prod.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}

resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = aws_vpc.prod.id
  service_name        = "com.amazonaws.us-east-1.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.prod.id
  service_name        = "com.amazonaws.us-east-1.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

### Encryption Strategy

```
Encryption at Rest:
  EBS volumes:   KMS CMK (customer-managed key)
  RDS/Aurora:    KMS CMK
  S3:            SSE-KMS with CMK
  EFS:           KMS CMK
  Secrets:       KMS CMK via Secrets Manager
  CloudTrail:    KMS CMK
  SQS/SNS:       KMS CMK

Encryption in Transit:
  ALB → EC2:     TLS 1.2+ (certificate via ACM)
  EC2 → RDS:     SSL enforced via parameter group
  MGN replication: TLS
  DMS replication: SSL
  Pod → AWS API:  TLS via HTTPS
```

### AWS Config Rules for Migration Compliance

```bash
# Deploy managed Config rules for migration security baseline
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "encrypted-volumes",
  "Source": {"Owner": "AWS", "SourceIdentifier": "ENCRYPTED_VOLUMES"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "rds-storage-encrypted",
  "Source": {"Owner": "AWS", "SourceIdentifier": "RDS_STORAGE_ENCRYPTED"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-public-read-prohibited",
  "Source": {"Owner": "AWS", "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "ec2-imdsv2-check",
  "Source": {"Owner": "AWS", "SourceIdentifier": "EC2_IMDSV2_CHECK"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "root-mfa-enabled",
  "Source": {"Owner": "AWS", "SourceIdentifier": "ROOT_ACCOUNT_MFA_ENABLED"}
}'
```

---

## 16. Cutover Strategies

A **cutover** is the moment when production traffic switches from the source (on-premises) to the target (AWS). The strategy determines the risk profile, rollback complexity, and downtime duration.

### Strategy 1 — Big Bang Cutover

```
Timeline: ─────────────────────────────────────────────────▶
                    │ CUTOVER WINDOW │
On-Prem:  ████████████▒▒▒▒▒▒▒▒▒▒▒▒▒░░░░░░░░░░░░░░░░░░░░
AWS:                                 ████████████████████
                   ↑                ↑
              App stopped        App started
                             (same or next day)
```

- **All-or-nothing** — entire application stack moves in one window
- **Downtime:** Hours (planned maintenance window, e.g., weekend 10PM–4AM)
- **Risk:** High — if AWS launch fails, full rollback to on-prem
- **When to use:** Small applications, non-critical workloads, hard business deadlines
- **Rollback:** Start application on source server, revert DNS

### Strategy 2 — Phased Migration

```
Phase 1: Static assets → S3 + CloudFront
Phase 2: Application servers → EC2 (keep DB on-prem via VPN)
Phase 3: DB migration (DMS full load + CDC running in parallel)
Phase 4: Final DB cutover (short window)
```

- **Incremental risk** — each phase is independently validated
- **Downtime:** Minutes (final DB cutover only)
- **Risk:** Medium — more complexity, but each phase is reversible
- **When to use:** Complex multi-tier applications with large databases

### Strategy 3 — Blue/Green Cutover (Zero Downtime)

```
Blue (On-Prem):  ████████████████████████▒▒▒▒▒▒░░░░░░░░░░░
Green (AWS):              ░░░░░░░░░░░░▒▒▒▒▒▒████████████████

Traffic routing via Route 53 weighted routing:
  T-0:   Blue=100%, Green=0%   (initial)
  T+1h:  Blue=90%,  Green=10%  (canary)
  T+4h:  Blue=50%,  Green=50%  (split)
  T+8h:  Blue=0%,   Green=100% (full shift)
  T+30d: Blue decommissioned
```

```bash
# Route 53 weighted routing for blue/green migration cutover
# Create blue (on-prem via VPN) record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "app.company.com",
          "Type": "A",
          "SetIdentifier": "blue-onprem",
          "Weight": 90,
          "TTL": 60,
          "ResourceRecords": [{"Value": "10.0.0.100"}]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "app.company.com",
          "Type": "A",
          "SetIdentifier": "green-aws",
          "Weight": 10,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "my-alb-123456.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'

# Shift traffic gradually: change weights
# Weight 90→0 (blue) and 10→100 (green)
```

### DNS TTL Strategy for Cutover

```bash
# 72 hours BEFORE cutover: lower TTL to 60 seconds
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"app.company.com","Type":"A","TTL":60,
    "ResourceRecords":[{"Value":"10.0.0.100"}]
  }}]}'

# During cutover: update A record to AWS (takes max 60 seconds to propagate)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"app.company.com","Type":"A","TTL":300,
    "AliasTarget":{"HostedZoneId":"Z35SXDOTRQ7X7K",
      "DNSName":"my-alb-123456.us-east-1.elb.amazonaws.com",
      "EvaluateTargetHealth":true}
  }}]}'
```

---

## 17. Post-Migration: Monitoring, Observability & Optimization

### CloudWatch Dashboards for Migrated Workloads

```bash
# Create a migration validation dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "MigrationValidation-Wave3" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "EC2 CPU Utilization",
          "metrics": [["AWS/EC2","CPUUtilization","InstanceId","i-12345678"]],
          "period": 60, "stat": "Average", "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "RDS Connections",
          "metrics": [["AWS/RDS","DatabaseConnections","DBInstanceIdentifier","mydb"]],
          "period": 60, "stat": "Sum"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "ALB 5xx Error Rate",
          "metrics": [["AWS/ApplicationELB","HTTPCode_Target_5XX_Count","LoadBalancer","app/my-alb/xxx"]],
          "period": 60, "stat": "Sum"
        }
      }
    ]
  }'

# Create alarms for P1 incidents post-migration
aws cloudwatch put-metric-alarm \
  --alarm-name "MigratedApp-HighErrorRate" \
  --alarm-description "HTTP 5xx error rate > 5% after migration" \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/my-alb/xxx \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:migration-oncall
```

### Cost Optimisation Post-Migration

```bash
# Get Savings Plans recommendations (after 1 week of actual usage data)
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option PARTIAL_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.{
    EstimatedSavings:SavingsPlansPurchaseRecommendationSummary.EstimatedSavingsAmount,
    SavingsPercentage:SavingsPlansPurchaseRecommendationSummary.EstimatedSavingsPercentage,
    HourlyCommitment:SavingsPlansPurchaseRecommendationSummary.HourlyCommitmentToPurchase
  }'

# Get Reserved Instance recommendations for RDS
aws ce get-reservation-purchase-recommendation \
  --service "Amazon RDS" \
  --payment-option PARTIAL_UPFRONT \
  --term-in-years ONE_YEAR \
  --lookback-period-in-days THIRTY_DAYS

# Tag all migrated resources for cost tracking
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list $(aws ec2 describe-instances \
    --filters "Name=tag:MigratedFrom,Values=onprem" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text | tr '\t' '\n' | \
    xargs -I{} echo "arn:aws:ec2:us-east-1:123456789012:instance/{}") \
  --tags Environment=production,MigrationWave=Wave3,CostCenter=CloudMigration
```

### AWS Compute Optimizer — Right-Sizing

```bash
# Export all over-provisioned EC2 recommendations to CSV
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'instanceRecommendations[*].{
    Instance:instanceArn,
    Current:currentInstanceType,
    Recommended:recommendationOptions[0].instanceType,
    MonthlySavings:recommendationOptions[0].estimatedMonthlySavings.value,
    Currency:recommendationOptions[0].estimatedMonthlySavings.currency
  }' \
  --output table > right-sizing-recommendations.txt

# Export recommendations to S3 for reporting
aws compute-optimizer export-ec2-instance-recommendations \
  --s3-destination-config bucket=my-optimization-reports,keyPrefix=ec2-recommendations/ \
  --fields finding,findingReasonCodes,utilizationMetrics,recommendationOptions \
  --include-member-accounts
```

---

## 18. Common Migration Pitfalls and How to Avoid Them

### Pitfall 1 — Skipping the Dependency Mapping

**Problem:** Migrating server A without knowing it has a hard dependency on server B that wasn't in scope for the wave. After cutover, application breaks because it can't reach B on-prem.

**Solution:** Run Application Discovery Service for 2-4 weeks before planning waves. Visualise dependencies in Migration Hub. Group all dependent servers in the same wave.

### Pitfall 2 — Not Right-Sizing for Cloud

**Problem:** Migrating a 16-core/64GB on-prem server to an equivalent EC2 instance (m5.4xlarge) and being surprised when the monthly bill is higher than on-prem. On-prem servers are typically utilised at 10-20% of peak capacity.

**Solution:** Use Migration Evaluator (actual utilisation data) for right-sizing, not nameplate specs. A 16-core server running at 15% average CPU probably fits on an m5.large (2 vCPU), not m5.4xlarge (16 vCPU).

### Pitfall 3 — Ignoring License Changes

**Problem:** Bringing Windows Server or SQL Server Standard to EC2 with BYOL on non-Dedicated Hosts violates the license agreement. Getting hit with a Microsoft audit.

**Solution:** Work with licensing team during Assess phase. For Windows/SQL Server BYOL, use Dedicated Hosts. Or convert to license-included EC2 (AWS handles Microsoft licensing).

### Pitfall 4 — No Rollback Plan

**Problem:** Cutover fails at 2AM, application is broken on AWS, source server is already shut down, on-call engineer panics.

**Solution:** During the 30-day validation period, source server must stay running (but idle). Rollback = start app on source + revert DNS. Document the rollback procedure in the runbook and test it on a non-critical application first.

### Pitfall 5 — Treating Migration as a One-Time Project

**Problem:** The migration "finishes" and the team disbands. Six months later the AWS costs are 40% higher than expected, no Savings Plans purchased, CloudWatch alerts triggering constantly, no one owns the observability.

**Solution:** Define cloud operating model (CloudOps team, SRE practices, FinOps processes) as part of the Mobilize phase. Migration is the beginning of cloud operations, not the end of a project.

### Pitfall 6 — DNS TTL Miscalculation

**Problem:** DNS record has TTL=3600 (1 hour). Cutover is executed, but users still hitting on-prem servers for up to an hour because their DNS cache hasn't expired.

**Solution:** Reduce TTL to 60 seconds 72+ hours before the cutover window. Restore TTL to 300-3600 after successful cutover.

### Pitfall 7 — Security Regression During Migration

**Problem:** To "make it work quickly," the team opens port 0-65535 on a security group or uses 0.0.0.0/0 ingress. These temporary rules become permanent.

**Solution:** Enforce security group review as part of the cutover checklist. Use AWS Config rule `restricted-ssh` and `restricted-common-ports` to flag violations automatically. Never allow `0.0.0.0/0` ingress rules in production security groups.

### Pitfall 8 — Forgetting Egress Costs

**Problem:** Application generating 10TB/month of outbound internet traffic. On-prem this was included in the DC contract. On AWS, data transfer out costs $0.09/GB = ~$900/month unexpected.

**Solution:** Model data transfer costs in the TCO analysis. Use CloudFront for static assets (cheaper egress). VPC endpoints eliminate data transfer costs for AWS API calls.

---

## 19. AI & New Trends in AWS Migration (2024–2025)

### 1. Amazon Q — Migration Acceleration (2024)

Amazon Q (the enterprise AI assistant) now has migration-specific capabilities:
- **Code transformation:** Amazon Q Developer's `q transform` automatically transforms Java 8/11 applications to Java 17 and upgrades Spring Boot versions — reducing refactoring effort by 60-80%
- **Migration Hub with AI recommendations:** Amazon Migration Hub now shows AI-generated remediation steps for discovered migration blockers
- **Q in the console:** Ask natural language questions about your migration status, cost projections, and optimization opportunities

```bash
# Amazon Q Code Transformation (Java modernisation)
# Run via CLI (replaces manual code upgrades for Java migrations)
aws codewhisperer start-code-analysis \
  --programming-language language=java \
  --scope FULL
# Q Developer identifies upgrade paths from Java 8 → Java 17
```

### 2. AWS Migration Hub Refactor Spaces (2023–2024)

Refactor Spaces is an AWS-managed strangler-fig pattern service that enables **incremental microservice extraction** from monoliths without re-writing the entire application:

```
Monolith on EC2                     Refactor Spaces
──────────────           →          ──────────────────────────────────
GET /orders   ────────────────────▶ Stays in monolith (default route)
POST /payment ──────────────────▶  New Lambda microservice (extracted)
GET /catalog  ──────────────────▶  New ECS microservice (extracted)

API Gateway + Refactor Spaces environment handles routing transparently
→ Monolith gradually hollows out as services are extracted
→ No DNS change, no client change
```

### 3. CloudEndure Migration → MGN Evolution

CloudEndure Migration was fully deprecated in 2024. AWS Application Migration Service (MGN) added:
- **Agent-less replication** for vSphere (2023) — no agent installation needed
- **MGN for EKS** — migrate containerised workloads directly to EKS
- **Post-launch validation automation** — custom scripts run after launch to validate application health

### 4. AWS Mainframe Modernisation (2023–2024)

AWS Mainframe Modernization service provides **automated COBOL → Java/microservices** conversion:
- Targets IBM z/OS, Micro Focus COBOL, PL/I applications
- Automated refactoring with AWS Blu Age (converts COBOL to Java)
- AWS MEND (Micro Focus Enterprise Developer) for manual migration
- Used by major banks and insurance companies migrating legacy COBOL systems

### 5. Graviton-First Migration Strategy (2024)

For replatforming workloads: migrate directly to **Graviton3 (ARM64) instances** rather than x86:
- 40% better price-performance than x86 equivalents
- Most modern software (JVM, Go, Python, Node.js, containers) runs on ARM64 without modification
- Aurora, ElastiCache, Lambda all support Graviton natively

### 6. FinOps During Migration (2024 Practice)

The rise of **FinOps (Cloud Financial Management)** as a discipline:
- Tagging automation via AWS Config and Tag Policies (org-level)
- **AWS Cost Anomaly Detection** with ML-based alerts (detects unexpected spend spikes post-migration)
- **Compute Optimizer + Cost Explorer integration** — actionable recommendations with estimated savings
- Savings Plans committed within 30-90 days of migration (after utilisation data is stable)

### 7. Infrastructure from Code (IfC) — CDK and AWS-Native IaC

While Terraform remains dominant, AWS CDK (Cloud Development Kit) is growing in migrations:
- Write infrastructure in TypeScript, Python, Java (not HCL)
- Higher-level constructs (e.g., `ApplicationLoadBalancedFargateService` in one line)
- Better for teams already skilled in application development languages

### 8. What to Learn Now for Cloud Migration Roles

- **AWS Application Migration Service (MGN)** — hands-on, set up a lab with a test server
- **AWS DMS + SCT** — practice an Oracle → Aurora PostgreSQL migration in a lab
- **Control Tower + Landing Zone Accelerator** — build a multi-account structure
- **Terraform** — write modules for VPC, EKS, and RDS from scratch
- **eksctl + Helm** — EKS cluster provisioning and application deployment
- **AWS FinOps** — Cost Explorer, Savings Plans, Compute Optimizer
- **AWS Migration Hub** — understand the portfolio tracking dashboard
- **Amazon Q Developer** — code transformation lab (Java upgrade)

---

## 20. Interview Summary & Key Concepts Cheat Sheet

### The 60-Second Migration Overview (Say This in an Interview)

> "AWS cloud migration is the process of moving an organisation's on-prem servers, databases, applications, and data to AWS. The typical methodology follows AWS's Migration Acceleration Program — three phases: Assess (discover and size the portfolio), Mobilize (build the Landing Zone, security baseline, IaC), and Migrate & Modernize (execute in waves).
>
> Every workload gets one of the 7 Rs: Retire it, Retain it on-prem, Rehost with MGN (lift & shift), Relocate to VMware Cloud, Replatform onto managed services like RDS, Repurchase as SaaS, or Refactor to cloud-native.
>
> As a DevOps engineer, my focus areas are: Landing Zone construction in Terraform, CI/CD pipeline setup, MGN replication for compute migrations, DMS for database migrations, EKS for container workloads, and post-migration monitoring and cost optimisation. Security is embedded from day one — GuardDuty, Security Hub, CloudTrail, KMS encryption, and SCPs to prevent security regression across all accounts."

### Key Numbers to Remember

| Fact | Value |
|---|---|
| MGN — typical cutover downtime | 10–60 minutes |
| DMS — supports engines | 20+ source, 15+ target engines |
| Backtrack window (Aurora MySQL) | Up to 72 hours |
| Control Tower — guardrails | 65+ mandatory + elective controls |
| Savings Plans — maximum discount | Up to 66% vs. on-demand |
| Reserved Instances — maximum discount | Up to 72% vs. on-demand |
| Direct Connect speeds | 1 Gbps, 10 Gbps, 100 Gbps |
| AWS regions — current count | 33 geographic regions (2024) |
| Transit Gateway — max VPC attachments | 5,000 per TGW |
| EKS — supported K8s versions | 4 minor versions at any time |

### Key Services → Migration Use Case Map

| AWS Service | Migration Use Case |
|---|---|
| MGN | Rehost servers (P2V, V2V) to EC2 |
| DMS | Replicate database data during migration |
| SCT | Convert Oracle/SQL Server schema to Aurora |
| DataSync | Move files (NFS/SMB → EFS/S3) |
| Snow Family | Move petabytes offline to S3 |
| Control Tower | Provision Landing Zone (multi-account) |
| Transit Gateway | Hub-and-spoke networking for all VPCs |
| Direct Connect | Private dedicated network to AWS |
| Migration Hub | Central tracking dashboard |
| Migration Evaluator | Right-sizing and TCO modelling |
| Velero | Kubernetes workload backup and migration |
| SCPs | Org-wide guardrails (prevent security regression) |
| Compute Optimizer | Right-size EC2/Lambda post-migration |
| Cost Explorer | Analyse and optimise cloud spend |

### Common Interview Follow-Up Questions on Migration

1. "How would you handle an application with no source code access during migration?"
   → Rehost via MGN. The agent works at the block level — no source code needed. Post-migration, treat it as a black box on EC2 with proper monitoring.

2. "What is the difference between DMS and MGN?"
   → MGN migrates entire servers (OS + apps + data) to EC2. DMS migrates database data only between database engines. They are complementary, not alternatives.

3. "How do you ensure a migration doesn't break compliance?"
   → Landing Zone with SCPs to enforce region restrictions, GuardDuty + Security Hub enabled before Wave 1, all data encrypted at rest with CMKs, CloudTrail enabled org-wide, compliance-as-code via Config rules.

4. "What is the strangler-fig pattern and when would you apply it in a migration?"
   → Gradually replace on-prem monolith functionality with cloud services while keeping the original running. Use AWS Refactor Spaces or API Gateway routing. Route new features to cloud; gradually move existing features.

5. "How would you handle database schema incompatibilities during an Oracle → Aurora migration?"
   → Run SCT to get an assessment report. Work through manual conversions for flagged objects (complex PL/SQL stored procedures). Use DMS for data replication. Run both Oracle and Aurora in parallel during cutover window. Validate row counts and spot-check data integrity before switching application connection string.

---

> **Next in this series:**
> - `AWS Migration-Patterns.md` — Deep-dive technical runbooks for each migration pattern with real-world scenarios
> - `AWS Migration-Security.md` — DevSecOps security controls, compliance frameworks (GDPR, PCI-DSS, ISO 27001), and WAF/GuardDuty configuration for migrated workloads

---

*References (2023–2025):*
- [AWS Migration Acceleration Program (MAP)](https://aws.amazon.com/migration-acceleration-program/)
- [AWS Cloud Adoption Framework v3 (2021)](https://docs.aws.amazon.com/whitepapers/latest/overview-aws-cloud-adoption-framework/welcome.html)
- [AWS Application Migration Service (MGN) docs](https://docs.aws.amazon.com/mgn/latest/ug/what-is-application-migration-service.html)
- [AWS DMS user guide](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html)
- [AWS Schema Conversion Tool user guide](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)
- [AWS Control Tower Landing Zone Accelerator (2023)](https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/)
- [Amazon Q Code Transformation (2024)](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/transform-overview.html)
- [AWS Well-Architected Migration Lens (2024)](https://docs.aws.amazon.com/wellarchitected/latest/migration-lens/migration-lens.html)
- [AWS Mainframe Modernization (2023)](https://aws.amazon.com/mainframe-modernization/)
- [FinOps Foundation + AWS Cost Management](https://www.finops.org/introduction/what-is-finops/)
