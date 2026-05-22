# AWS Migration — Interview Guide
## Hiring Perspective: Senior DevOps Engineer (3 Years Experience)

> **How to read this file:** This document is written from the **interviewer's perspective** — what a hiring manager at a cloud-focused AWS partner (like the JD company) would ask, check, and assess when evaluating a DevOps/DevSecOps engineer for an AWS migration delivery role. Every question is followed by the **ideal answer** so you know exactly what the interviewer expects to hear.

---

## The Hiring Manager's Mindset

Before the first question is asked, an interviewer for this specific role is running one question through their head:

> *"Can this person land in a client's data center at week one, understand a 200-server environment, start architecting a migration plan in Terraform, and be on-call for the production cutover — without holding my hand?"*

At 3 years experience, the bar is:
- **Not expected:** To have led a 1,000-server enterprise migration solo
- **Expected:** To have hands-on AWS experience, solid Terraform, have touched containers, and have enough judgment to debug a failing DMS task at 2AM

The interviewer is looking for three things in every answer:
1. **Precision** — do they use the right service names, right terminology?
2. **Depth** — can they go from concept to CLI to trade-off?
3. **Ownership** — do they say "I did X" or "the team did X and I watched"?

---

## Interview Structure (Typical Format for This Role)

```
Round 1 — HR/Recruiter Screen (30 min)
  └── Culture fit, motivation, logistics, basic AWS knowledge check

Round 2 — Technical Phone Screen (60 min)
  └── AWS fundamentals, Terraform, containers — eliminates ~50% of candidates

Round 3 — Technical Deep-Dive (90–120 min)
  └── Migration-specific scenarios, architecture, live problem-solving
  └── Sometimes: take-home Terraform task reviewed in session

Round 4 — Architecture / Whiteboard (60 min)
  └── Design a migration architecture from scratch
  └── Walk through your past project in detail

Round 5 — Final (Culture/Leadership) (30–45 min)
  └── Values alignment, client-facing scenarios, career goals
```

---

## Section 1 — Qualification Checklist (Before Hiring Decision)

This is the **interviewer's scoring sheet** — every box must be checked for a hire:

### Hard Requirements (Must Have — Automatic Reject if Missing)

- [ ] **4+ years DevOps/Cloud experience** (or 3+ with exceptional depth)
- [ ] **AWS — practical hands-on** (not just theory — can describe real infrastructure they built)
- [ ] **Terraform — wrote modules from scratch** (not just copy-pasted examples)
- [ ] **CI/CD pipeline — designed and operated one** (GitLab CI, GitHub Actions, Jenkins, CodePipeline)
- [ ] **Containers — EKS or ECS production experience** (deployed real workloads, not just tutorials)
- [ ] **Linux proficiency** — comfortable on the command line, systemd, networking tools
- [ ] **Understands migration fundamentals** — at minimum knows MGN, DMS, Landing Zone concepts

### Strong-to-Have (Differentiator — Not Automatic Reject)

- [ ] AWS certification (Solutions Architect Associate or Professional, DevOps Professional)
- [ ] Direct experience with DMS or MGN in a real migration project
- [ ] Control Tower / multi-account architecture experience
- [ ] SRE principles (SLO/SLA/error budgets, monitoring, alerting, chaos engineering)
- [ ] Database migration experience (schema conversion, cutover planning)
- [ ] Customer-facing or consulting experience
- [ ] Scripting (Python or Bash automation)
- [ ] Knows DevSecOps practices (GuardDuty, Security Hub, secrets management)

### Behavioral Signals the Interviewer Watches For

- [ ] **Ownership mindset** — says "I built" not "we had a system that"
- [ ] **Failure stories** — can describe a migration failure and what they learned (shows real experience)
- [ ] **Asks clarifying questions** during technical scenarios (shows engineering judgment)
- [ ] **Trade-off awareness** — doesn't give a single "correct" answer but acknowledges alternatives
- [ ] **Client communication** — can explain a technical decision in business terms

---

## Section 2 — HR / Recruiter Screen Questions

These are lightweight questions that filter for basic fit and eliminate time-wasters before the technical rounds.

---

**Q1: Tell me about your experience with AWS cloud migrations specifically — not general AWS, but moving something from on-prem or another cloud to AWS.**

> **What the interviewer is listening for:**
> - Specific project (not generic)
> - Their personal role (were they hands-on or observing?)
> - Scale (number of servers, DB size, timeline)
> - Which tools (MGN? DMS? Manual?)

**Model Answer:**
> "At my last role, I was part of a 6-person team migrating 80 on-prem servers for a logistics company — they had a data center lease expiring. My specific responsibility was the compute migration: I set up AWS Application Migration Service, installed the MGN agents on the Linux servers, configured the replication settings, and ran the test launches. I personally executed cutovers for 3 waves totalling about 40 servers. The DBAs handled the Oracle-to-RDS migration in parallel, though I helped configure the DMS replication monitoring and wrote the CloudWatch alarms for replication lag. The whole project ran over 5 months and we decommissioned the DC on schedule."

**Red flag answer:** "We moved our company's infrastructure to AWS" (vague, no tools, no scale, no personal contribution).

---

**Q2: What is your comfort level with Terraform — can you write a module from scratch?**

> **What the interviewer is listening for:**
> - Can they distinguish between writing modules vs. using them?
> - Do they know state, remote backends, workspaces, import?
> - Have they dealt with real problems (state corruption, drift)?

**Model Answer:**
> "Yes — I've written reusable Terraform modules for VPC, EKS cluster, RDS Aurora, and IAM roles. I structure modules with variables, outputs, and a clear interface so teams can consume them without knowing internals. I've also dealt with state issues in team environments — we use S3 backend with DynamoDB locking and I've had to do `terraform import` to bring manually-created resources under management. One time a colleague applied directly from their laptop and we had a state conflict — after that we enforced all applies go through our GitLab CI pipeline so the state is always authoritative."

---

**Q3: What CI/CD tools have you used and what's the most complex pipeline you've built?**

**Model Answer:**
> "Primarily GitLab CI and GitHub Actions. The most complex pipeline I built was a multi-stage pipeline for deploying microservices to EKS: it ran unit tests, built and pushed the Docker image to ECR with a git-sha tag, ran Trivy to scan the image for CVEs, updated the Helm values file with the new image tag, committed that back to the GitOps repository, and Argo CD picked up the change and deployed to the cluster. If the CVE scan found a CRITICAL severity finding, the pipeline failed and created a Jira ticket automatically via webhook. The whole pipeline ran in under 8 minutes."

---

**Q4: Why do you want to work on migration projects specifically? Many DevOps engineers prefer greenfield work.**

> **What the interviewer is listening for:**
> - Is this person genuinely interested in migrations or just needs a job?
> - Do they understand what migration delivery looks like (client-facing, deadline pressure, high stakes)?

**Model Answer (show genuine reasoning):**
> "I actually prefer migrations because the problems are concrete and the impact is immediate and measurable. Greenfield is great but you're often building for hypothetical future scale. In a migration, you're dealing with real production systems, real business constraints, real deadlines — and when you complete a cutover successfully at 2AM and the application comes up clean, it's genuinely satisfying. I also find the variety interesting — every migration has a different stack, different legacy debt, different risk profile. You can't use a playbook blindly; you have to adapt."

---

## Section 3 — Technical Phone Screen Questions (Medium Level)

These questions separate people who've read the AWS docs from people who've actually done it.

---

**Q5: What is AWS Application Migration Service (MGN) and how does it differ from the old AWS Server Migration Service (SMS)?**

> **Depth expected:** Know that SMS is deprecated, know how MGN works technically, know the key advantages.

**Model Answer:**
> "AWS Server Migration Service was deprecated in early 2022 and replaced by AWS Application Migration Service, or MGN. The fundamental difference is the replication approach: SMS did VM snapshot-based replication — it took periodic snapshots (every few hours) and uploaded them to AWS, which meant you had to plan around snapshot windows and accept longer RTO. MGN uses continuous block-level replication — the agent installed on the source server mirrors every disk write to a replication server in your AWS account in near real-time. This means at cutover time the EC2 instance is fully up-to-date (lag typically under 60 seconds), so the cutover downtime window is just 10-60 minutes rather than hours. MGN also supports both Windows and Linux, has better API support for automation, and integrates with Migration Hub for tracking."

---

**Q6: Walk me through what happens — technically — when you initiate a cutover in MGN. Step by step.**

> **Depth expected:** This is a precision question. They want to see you know the exact sequence, not just "it migrates the server."

**Model Answer:**
> "When you initiate a cutover, MGN: one — does a final sync of any outstanding block-level changes from the source to the staging EBS volumes. Two — converts the staging EBS volumes to a bootable state, adjusting boot configuration for the AWS hypervisor (virtio drivers for Linux, AWS PV drivers for Windows). Three — launches the cutover EC2 instance from those EBS volumes using the launch template you configured (instance type, subnet, security groups, IAM profile). Four — the instance boots and runs any post-launch actions you've configured (scripts, SSM documents). Five — you validate the instance in AWS. Six — you call `finish-cutover` in the console or CLI, which marks replication as complete and stops the MGN agent communication. After that, you have a 30-day window before you decommission the source server. The source server remains running throughout unless you manually stop it."

---

**Q7: What is a Landing Zone in AWS, and what are the minimum components it must have?**

> **Depth expected:** Multi-account, Control Tower, security baseline — not just "a VPC."

**Model Answer:**
> "A Landing Zone is a secure, well-governed, multi-account AWS environment that serves as the foundation for all cloud workloads. The minimum components are: AWS Organizations to manage accounts hierarchically, at least four foundational accounts — management account for billing and org-level policies, a log archive account for centralised audit logs, a security tooling account for GuardDuty and Security Hub aggregation, and one or more workload accounts. Service Control Policies attached to OUs to enforce guardrails — like denying API calls outside approved regions or preventing CloudTrail from being disabled. IAM Identity Center for single sign-on across accounts. An org-level CloudTrail trail writing to an S3 bucket in the log archive account. AWS Config enabled in every account with an aggregator in the security account. GuardDuty enabled and delegated to the security account. Default VPC deleted from new accounts. That's the minimum. AWS Control Tower automates most of this, but I've also built it manually with Terraform using the Landing Zone Accelerator pattern."

---

**Q8: You're running DMS to replicate a MySQL database to Aurora. The replication task shows a lag of 45 minutes. How do you diagnose and fix it?**

> **Depth expected:** They want to see structured troubleshooting, knowledge of DMS architecture, specific metrics.

**Model Answer:**
> "I'd start by checking two things in parallel: the DMS replication instance's CPU and memory in CloudWatch, and the source MySQL server's replication metrics. 45 minutes of lag usually means the DMS SQL thread is falling behind applying changes.

> First, check the replication instance: if CPU is at 100%, the instance is undersized — scale up the replication instance class. If memory is high, same fix.

> Second, on the source MySQL, run `SHOW PROCESSLIST` and `SHOW MASTER STATUS` to see if there are long-running transactions blocking the binlog from advancing. A single long transaction can hold up replication.

> Third, check the DMS task logs for errors — a table that fails to replicate silently can cause lag to accumulate. Look for `ERROR` entries in CloudWatch Logs for the task.

> Fourth, check the source server's I/O — if binlog is writing to a slow disk, the I/O thread can't read fast enough.

> Fixes in order of likelihood: scale up the replication instance if CPU-bound, enable parallel apply threads in the DMS task settings if the schema supports it, resolve long-running transactions on the source, or split the task into multiple tasks by table range if it's a single large table causing the bottleneck.

> If the lag is persistent and the cutover is tomorrow, I'd consider stopping non-critical writes to the source temporarily to let the lag drain, then resume."

---

**Q9: What is the difference between AWS Direct Connect and a Site-to-Site VPN? When would you use each for a migration project?**

**Model Answer:**
> "Direct Connect is a physical, dedicated private network connection between your on-premises environment and AWS — you order a port through an AWS Direct Connect partner and get a dedicated 1, 10, or 100 Gbps connection that doesn't go over the public internet. The latency is consistent and the bandwidth is guaranteed. The downside is it takes weeks to months to provision and costs more — there's a port-hour fee plus data transfer.

> Site-to-Site VPN is an IPSec-encrypted tunnel over the regular internet — provisioned in minutes, much cheaper, but bandwidth is limited (about 1.25 Gbps per tunnel) and latency varies with internet conditions.

> For a migration project: I'd use VPN first because I can set it up immediately while the Direct Connect circuit is being ordered. Once Direct Connect is provisioned, I'd use it as the primary path for MGN replication and DMS data replication — you're potentially pushing terabytes of data and you need the bandwidth and consistency. VPN stays as a backup path (BGP failover). For smaller migrations under 5TB, VPN alone is usually sufficient. I'd never rely on VPN alone for a large database migration where the replication traffic would saturate the bandwidth."

---

**Q10: Explain IRSA in EKS — what problem does it solve and how does it work mechanically?**

> **Depth expected:** Know the OIDC flow, not just "it lets pods access AWS services."

**Model Answer:**
> "IRSA stands for IAM Roles for Service Accounts. It solves the problem of giving pods in EKS access to AWS services (S3, Secrets Manager, DynamoDB) without hard-coding AWS credentials or putting them in environment variables.

> Mechanically, it works like this: when you create an EKS cluster, you associate an OIDC identity provider with it. This OIDC provider issues short-lived JWT tokens to pods. When a pod starts, the projected service account token is mounted as a file inside the pod. The pod's AWS SDK detects this token and calls `sts:AssumeRoleWithWebIdentity` — it presents the JWT to AWS STS and says 'I am this Kubernetes service account, I want to assume this IAM role.' AWS STS validates the JWT signature against the OIDC provider's public keys, checks that the `sub` claim in the JWT matches the IAM role's trust policy condition, and if everything checks out, returns temporary credentials (Access Key + Secret + Session Token) valid for 1 hour. The SDK uses those credentials for all subsequent API calls, and they auto-refresh before expiry.

> The IAM role's trust policy looks like this: it allows `sts:AssumeRoleWithWebIdentity` from the OIDC provider, conditioned on the `sub` claim matching `system:serviceaccount:my-namespace:my-service-account`. This means only pods running with that specific Kubernetes service account in that specific namespace can assume the role.

> The alternative before IRSA was attaching IAM policies directly to the EC2 node's instance profile — but that gave every pod on the node the same permissions, which is a massive blast radius. IRSA gives per-pod least-privilege."

---

**Q11: What is a Service Control Policy (SCP) and how is it different from an IAM policy?**

**Model Answer:**
> "An SCP is a policy attached to an AWS Organizations Organizational Unit or account that sets the maximum permission boundary for everyone in that account — including the root user. It doesn't grant permissions; it restricts them. If an SCP denies an action, no IAM policy in that account can override it.

> An IAM policy grants or denies permissions for a specific principal — a user, role, or service. IAM policies operate within the account.

> The key difference: IAM policies work at the principal level, SCPs work at the account level as a ceiling. Example: if an SCP denies `ec2:RunInstances` outside `eu-central-1`, then even if an IAM Administrator in that account tries to launch an EC2 in `us-east-1`, the action is denied. No IAM policy can grant what an SCP denies.

> In a migration Landing Zone, SCPs are essential guardrails: prevent accounts from leaving the organization, prevent disabling CloudTrail or GuardDuty, restrict resources to approved regions, prevent creation of public S3 buckets, enforce encryption. We set these at the OU level so every new account automatically inherits them."

---

## Section 4 — Technical Deep-Dive Questions (Medium-Hard)

These are the questions that reveal whether someone has real production experience or has only studied for interviews.

---

**Q12: Design a multi-account AWS Landing Zone for a company with 200 employees, 3 environments (dev/staging/prod), and compliance requirements (ISO 27001). Draw the account structure and explain every decision.**

> **What the interviewer is assessing:** Can they make real architectural decisions? Do they know WHY multi-account, not just WHAT?

**Model Answer:**

> "I'd start with the account structure. I need a minimum of these accounts:

> - **Management account** — only for billing, organization management, and SCPs. No workloads deployed here. Strictly controlled — only 2-3 people should have access.
> - **Log Archive account** — all CloudTrail logs, AWS Config snapshots, VPC Flow Logs, S3 access logs from every account go here. S3 bucket with Object Lock (WORM) so logs are tamper-proof — ISO 27001 requirement. No one can delete these logs, even account admins.
> - **Security Tooling account** — GuardDuty delegated administrator, Security Hub administrator, IAM Access Analyzer organization analyzer, AWS Config aggregator. The security team monitors from here across all accounts.
> - **Network account** — owns the Transit Gateway, Direct Connect, and Route 53 Resolvers. VPCs in other accounts attach to the TGW here. Centralised ingress/egress.
> - **Shared Services account** — internal tools: Managed Active Directory, Nexus/JFrog Artifactory, any shared DNS zones, VPN concentrators.
> - **Dev, Staging, Prod accounts** — one per environment. Why separate accounts, not separate VPCs in one account? Blast radius isolation — a compromised dev credential can't affect prod. Clean billing per environment. Service limit isolation.
> - **Sandbox accounts** — each developer gets a personal sandbox account with a budget alarm. Separate from dev so experimental work can't affect shared dev environment.

> All accounts provisioned via Control Tower Account Factory — ensures they're born with the right baselines (GuardDuty on, CloudTrail on, default VPC deleted, SSO configured).

> SCPs at the root level: deny leaving organization, deny disabling security services. SCPs at the Prod OU level: deny creating internet gateways without approval, deny public S3 ACLs, require MFA for console access, restrict to EU regions only (if data residency is required for ISO 27001).

> For ISO 27001 specifically: every API call logged (CloudTrail), encryption enforced via Config rules, access reviews quarterly via IAM Access Analyzer, patch compliance tracked via Systems Manager Patch Manager, and Security Hub with AWS Foundational Security Best Practices standard enabled."

---

**Q13: You're about to cut over a 3-tier application (ALB + 4 EC2 app servers + RDS MySQL) from on-premises to AWS. It's 11PM on a Sunday. Walk me through what you do in the 2 hours before, during the cutover, and the 30 minutes after.**

> **What the interviewer is assessing:** Do they have a real runbook in their head? Do they think about rollback? Do they think about communication?

**Model Answer:**

> "Two hours before:

> I confirm DMS replication lag is under 60 seconds — if it's above that, I do not proceed, we reschedule. I run the pre-cutover checklist: test that the EC2 instances pass health checks, test-launch already done and validated this week, ALB target group shows all targets healthy, RDS is accessible from the app servers via security group, SSL certificate in ACM is valid, DNS TTL has been reduced to 60 seconds since Thursday. I notify the on-call team in Slack with the cutover start time and the rollback trigger condition. I open the rollback document so it's one click away.

> During cutover:

> I put the application in maintenance mode — show a maintenance page via ALB. I confirm DMS lag hits zero seconds and stop the replication task. I immediately run the data validation script — row count comparison on the top 10 tables between on-prem MySQL and Aurora. If any count is off by more than 0.01%, I stop and investigate before proceeding.

> Assuming validation passes: I update the application's DB connection strings in AWS Secrets Manager to point at Aurora. I restart the application services on the 4 EC2 instances. I run smoke tests — automated test suite that hits 15 critical API endpoints and checks response codes and response times. I verify the ALB shows all 4 targets healthy. I update the DNS A record for `app.company.com` to the ALB DNS name.

> Thirty minutes after:

> I watch CloudWatch — CPU on EC2 instances, DatabaseConnections on Aurora, ALB 5xx error rate, ALB response time P99. I set up a 30-minute alert window where any 5xx rate above 2% triggers an immediate call. I check application error logs in CloudWatch Logs for anything new. I confirm with the application team lead that user-facing functionality is working.

> Rollback condition: if 5xx rate is above 5% for more than 2 minutes, or a critical data integrity issue is found — I revert the DNS record to point at the on-prem load balancer (60-second TTL means it takes effect quickly), restart the on-prem app servers, and revert DB connection strings. I then declare the cutover failed, write up the post-mortem, and reschedule.

> After 30 minutes clean — I post the success notification in Slack, increase DNS TTL back to 300, and hand off to the day team for the 30-day validation monitoring period."

---

**Q14: What is Terraform state and why is it critical in a team migration environment? What can go wrong and how do you prevent it?**

**Model Answer:**

> "Terraform state is a JSON file that maps the resources defined in your `.tf` files to the real infrastructure in AWS. When you run `terraform plan`, Terraform reads the state to know what currently exists, compares it to what your code defines, and outputs the diff. Without state, Terraform would try to create everything from scratch every time.

> In a team environment, state management is critical because:

> Problem 1 — **Concurrent applies**: if two engineers run `terraform apply` simultaneously, they'll both read the same state, both plan their changes, and both try to apply — resulting in conflicting modifications and a corrupted state. The fix is DynamoDB state locking: before any apply, Terraform acquires a lock in a DynamoDB table. The second apply waits or fails until the lock is released.

> Problem 2 — **State in the wrong place**: if state is stored locally, the next person on the team can't see what's deployed. Store state in S3 with versioning enabled and encryption (KMS). This gives you a history of all state changes and the ability to roll back to a previous state version if corruption occurs.

> Problem 3 — **Manual changes creating drift**: someone creates a security group manually in the console during an incident. Now Terraform's state doesn't match reality. Next `terraform plan` either tries to delete the resource or errors. Fix: enforce IaC discipline (no console changes), use `terraform import` to bring manual resources under management, and run `terraform plan` in CI on every PR to detect drift early.

> Problem 4 — **State contains sensitive data**: RDS passwords, private keys. Use Secrets Manager or SSM Parameter Store for secrets rather than putting them directly in resource definitions. Enable S3 bucket encryption for the state bucket.

> For a migration project specifically, I'd separate state by environment and by layer — one state file for networking, one for compute, one for databases. This limits the blast radius of a state corruption event and allows teams to work in parallel on different layers."

---

**Q15: Explain the difference between EKS managed node groups, self-managed node groups, and Fargate. When would you choose each in a migration project?**

**Model Answer:**

> "Managed node groups: AWS manages the EC2 instances in the node group — it handles AMI updates, node termination, draining, and replacement. You define the configuration (instance type, min/max size, labels, taints) and AWS provisions and manages the underlying EC2s. You still have access to the nodes for debugging. Cost: standard EC2 pricing.

> Self-managed node groups: you manage the EC2 instances yourself — you're responsible for the AMI (ideally a custom golden AMI), the ASG configuration, the launch template, and node upgrades. More control, more operational burden. Use case: when you need specific kernel parameters, custom security agents, or GPU instance types not yet supported by managed node groups.

> Fargate: serverless — no nodes at all. You define the pod spec and AWS runs it on shared infrastructure. You pay per vCPU-second and GB-second actually consumed. No node patching, no cluster capacity planning. The limitation is no DaemonSets (Fargate pods are isolated), no privileged containers, and slightly higher per-unit cost than EC2 for sustained workloads. Limited storage options (20GB ephemeral, no hostPath).

> In a migration context: I'd use managed node groups as the default for migrated containerised workloads — it's the right balance of control and managed operations. Self-managed for any workload that needs specific host-level configuration (e.g., an app that requires a specific kernel version or needs to mount /dev devices). Fargate for bursty, event-driven workloads or for the migration tooling itself (Velero, migration agents running as pods) where I don't want to size nodes just for them.

> One more option worth mentioning: Karpenter — AWS's open-source node provisioner that replaces Cluster Autoscaler. Karpenter provisions exactly the right instance type for each pod's resource request in seconds, rather than scaling pre-defined node groups. In 2024 this is the recommended approach for production EKS."

---

**Q16: How would you migrate a containerised application that currently runs on Docker Swarm to Amazon ECS?**

> **Depth expected:** Know the conceptual mapping, the concrete steps, and the gotchas.

**Model Answer:**

> "Docker Swarm and ECS have a conceptual mapping: Swarm stacks map to ECS services, Swarm services map to ECS task definitions, Docker Compose files can be partially converted to ECS task definitions using `ecs-cli compose`.

> Practical steps:

> Step 1 — **Assess the current Swarm setup**: document all services, their image sources, port mappings, environment variables, secrets, volumes (local vs. NFS), network topology, health check commands, and resource constraints.

> Step 2 — **Container images**: if images are in a private Swarm registry, push them to ECR. Tag with semantic versions, not just `latest`. Enable ECR image scanning.

> Step 3 — **Secrets**: Swarm secrets (`docker secret create`) map to ECS Secrets backed by Secrets Manager or SSM Parameter Store. Update task definitions to reference secrets by ARN, not env vars.

> Step 4 — **Persistent storage**: Swarm volumes on NFS map to EFS volumes mounted in ECS tasks via EFS volume configurations in the task definition.

> Step 5 — **Task definition**: write the ECS task definition (JSON or Terraform `aws_ecs_task_definition`). Map each Swarm service to a task definition with the right CPU/memory limits, environment variables, secrets, log configuration (awslogs driver → CloudWatch Logs), and health check.

> Step 6 — **ECS Service**: deploy as an ECS service with the desired count and an ALB target group. If Swarm had multiple services that talk to each other, use ECS Service Connect (2022 feature) or Cloud Map for service discovery — this replaces Swarm's built-in DNS service discovery.

> Step 7 — **Validate**: run smoke tests against the ECS deployment in staging before cutting over production traffic from the Swarm cluster.

> Main gotcha: Swarm's built-in overlay networking is replaced by AWS VPC networking in ECS. If services used overlay network IPs to communicate, replace those with ALB listener rules, Cloud Map service DNS names, or ECS Service Connect endpoints."

---

**Q17: A pod in your EKS cluster is getting a `403 Access Denied` error when trying to read from an S3 bucket. Debug this step by step.**

> **What the interviewer is assessing:** Systematic IAM debugging in EKS. Not just "check the IAM role."

**Model Answer:**

> "Step 1 — **Confirm the error source**: exec into the pod and run `aws s3 ls s3://my-bucket/ 2>&1`. Confirm the exact error message. Is it `Access Denied`, `NoCredentialProviders`, or `InvalidClientTokenId`? Each points to a different problem.

> If `NoCredentialProviders`: the pod has no AWS identity at all — either IRSA is not configured (no service account annotation) or the OIDC provider is not enabled on the cluster.

> If `Access Denied` (403): the pod has an identity but it lacks the permission. Most common path.

> Step 2 — **Check the identity**: in the pod, run `aws sts get-caller-identity`. This tells you exactly which IAM role or user the pod is authenticated as. If it returns the node's EC2 instance role rather than a dedicated pod role — IRSA is misconfigured.

> Step 3 — **Verify IRSA annotation on the service account**: `kubectl describe serviceaccount app-sa -n production` — look for the annotation `eks.amazonaws.com/role-arn`. If it's missing or wrong ARN, add/fix it.

> Step 4 — **Check the IAM role trust policy**: the role's trust policy must allow `sts:AssumeRoleWithWebIdentity` from the cluster's OIDC issuer, with a condition on `StringEquals` matching `system:serviceaccount:production:app-sa`. If the namespace or service account name doesn't match exactly — denied.

> Step 5 — **Check the IAM permissions policy**: use IAM Policy Simulator or `aws iam simulate-principal-policy` to test whether the role has `s3:GetObject` on `arn:aws:s3:::my-bucket/*`. Confirm the bucket name and prefix are correct in the policy.

> Step 6 — **Check the S3 bucket policy**: the bucket might have an explicit Deny that overrides the role's Allow. Explicit Deny always wins. Check `aws s3api get-bucket-policy --bucket my-bucket` for any `Deny` statements.

> Step 7 — **Check Block Public Access and VPC endpoint policy**: if there's an S3 VPC endpoint with a restrictive policy, it might be denying the request even if the IAM role has permission. Check `aws ec2 describe-vpc-endpoints` and review the endpoint policy.

> Most common root causes in order: wrong role ARN in service account annotation, trust policy condition doesn't match the actual namespace/SA name, IAM policy missing the `/*` on the bucket ARN (grants ListBucket on `arn:aws:s3:::bucket` but requires `arn:aws:s3:::bucket/*` for GetObject/PutObject)."

---

**Q18: What is the Strangler Fig pattern and how would you apply it to migrate a monolith to microservices on EKS during a cloud migration?**

**Model Answer:**

> "The Strangler Fig pattern — named after the fig tree that slowly wraps around and replaces a host tree — is an incremental migration strategy where you extract functionality from a monolith one piece at a time, replacing it with a new service, until eventually the monolith is empty and can be decommissioned.

> Applying it to a migration: instead of a risky big-bang rewrite, you introduce an API Gateway or a proxy (AWS API Gateway, ALB, or Nginx reverse proxy) in front of the monolith. Initially 100% of traffic goes to the monolith. You extract one capability — say, the `POST /payments` endpoint — into a new microservice deployed on EKS. You configure the proxy to route `/payments` to the new EKS service, while all other paths still go to the monolith. The new service talks to its own Aurora database. You validate the new service works, then extract the next capability.

> The key is: the client (browser, mobile app, other services) sees the same hostname and the same API contract — they don't know anything changed. The only change is inside the routing layer.

> AWS Refactor Spaces (launched 2022) is an AWS-managed implementation of this pattern — it creates an environment with an Application Router that handles the routing automatically, so you don't have to manage the proxy yourself.

> When to use it in a migration: when the client wants to leave on-prem AND modernise at the same time. The monolith is first rehosted to EC2 (fast, low risk), then the Strangler Fig extracts services incrementally to EKS over the following months. This separates the DC exit timeline from the modernisation timeline — the DC is emptied on schedule, modernisation continues at a sustainable pace."

---

**Q19: How do you implement zero-downtime deployments for a migrated application on EKS?**

**Model Answer:**

> "Several mechanisms depending on the risk tolerance:

> **Rolling update (default)**: Kubernetes replaces pods incrementally. Configure `maxSurge: 25%` and `maxUnavailable: 0` in the Deployment spec — this ensures at least the current number of pods is always running while new pods start up. Downside: both old and new versions run simultaneously, so APIs must be backward-compatible.

> **Blue/Green with Argo CD or Argo Rollouts**: deploy a completely new version alongside the current one (blue=current, green=new). Switch traffic at the ALB/Ingress level when the green deployment passes health checks. If something goes wrong, revert the traffic switch in seconds. Argo Rollouts automates this pattern.

> **Canary deployment**: route a small percentage of traffic (say 5%) to the new version. Monitor error rates and latency. Gradually increase to 100% if metrics are good, abort and rollback if not. Argo Rollouts supports canary with Prometheus-based metric analysis to automate the promotion decision.

> **PreStop hook + terminationGracePeriodSeconds**: critical for zero-downtime — when a pod is being terminated, Kubernetes removes it from the service endpoint list and then sends SIGTERM. Without a `preStop` hook, there's a race condition where requests can hit a terminating pod. Add a `preStop` sleep of 5-10 seconds to give the load balancer time to drain connections before the pod stops accepting new ones.

```yaml
lifecycle:
  preStop:
    exec:
      command: ['/bin/sh', '-c', 'sleep 10']
terminationGracePeriodSeconds: 60
```

> For a migration project, I'd start with rolling updates (simplest) and add Argo Rollouts canary once the application is stable on EKS."

---

**Q20: How do you handle secret rotation in a migrated application on AWS — specifically for database credentials?**

**Model Answer:**

> "The goal is automatic rotation with zero application downtime. AWS Secrets Manager + RDS native integration handles this well.

> When Secrets Manager rotates an RDS/Aurora credential, it uses a Lambda rotation function that: 1 — creates a new password and updates the DB user's password in the database. 2 — stores the new password in Secrets Manager. 3 — marks the old secret version as `AWSPREVIOUS`. During the brief window between the DB password change and the secret update completing, the application might try to connect with the old password — this is handled by the rotation function using the `AWSPREVIOUS` version as a fallback.

> On the application side: instead of caching the secret at startup, the application should fetch it fresh from Secrets Manager on each connection or use a short cache TTL (60 seconds). AWS SDK's Secrets Manager client also has built-in caching with refresh.

> For a newly migrated application that hard-coded DB credentials in a config file — the migration is the perfect time to fix this: move the credentials into Secrets Manager, update the application to call Secrets Manager at startup, enable 30-day auto-rotation.

> For Kubernetes specifically: External Secrets Operator syncs secrets from Secrets Manager into Kubernetes Secrets automatically. When Secrets Manager rotates, the Kubernetes Secret is updated, and with the `refreshInterval` set to 60 seconds, the pod sees the new credential within 1 minute without a restart."

---

## Section 5 — Architecture / Scenario Questions (Hard)

These are open-ended problems where there's no single right answer — the interviewer is evaluating your reasoning process.

---

**Q21: SCENARIO — You are 3 weeks into a migration project. 40 servers are being replicated via MGN. Your AWS Direct Connect circuit goes down at 9AM on a Tuesday. What do you do?**

> **What the interviewer is assessing:** Incident management under pressure. Know the failover path. Business communication. Don't panic.

**Model Answer:**

> "First — assess impact, not panic. Direct Connect going down affects: MGN replication (replication lag will start climbing), DMS CDC replication (if running), any application traffic routing through Direct Connect (if any already cut over).

> Immediate actions (first 15 minutes):

> Call the Direct Connect provider's NOC to report the outage and get an ETA — is this minutes, hours, or days? Simultaneously, verify that the BGP failover to the backup Site-to-Site VPN has activated automatically (if we set it up correctly during Mobilize phase). Check the BGP routing tables on the customer gateway.

> If VPN failover is working: the blast radius is reduced. MGN will continue replicating over VPN at lower bandwidth. Replication lag will increase — check MGN console. If lag stays under 4 hours, we're in a manageable state for a short outage.

> Communication: post a status update in the project Slack channel within 15 minutes. Tell the project manager, client contact, and anyone scheduled for a cutover today that we're pausing all cutovers until connectivity is restored. Do NOT proceed with a cutover with degraded connectivity — if the cutover instance needs to pull data from the replication server and the link is slow, the cutover will take far longer than planned.

> If VPN failover is NOT working: higher urgency. Check the CGW configuration, check IPSec tunnel status (`aws ec2 describe-vpn-connections`). If VPN is also down — we've lost connectivity entirely. At this point, no replication is happening. Check if there are any cutovers that already completed and applications running in AWS — if those depend on on-prem via the DC link, they may be broken. Prioritise restoring connectivity.

> Root cause and post-incident: once connectivity is restored, sync MGN and wait for lag to drain before resuming any cutovers. Write an incident report. Consider whether the Direct Connect architecture needs improvement (redundant ports, dual carriers) for the remainder of the project."

---

**Q22: SCENARIO — The client's CISO comes to you two weeks before the first production wave and says: 'We need to prove to our auditors that no production data will ever leave our EU data centers during the migration.' How do you address this?**

> **What the interviewer is assessing:** Data residency, encryption in transit, AWS region control, audit evidence.

**Model Answer:**

> "This is a legitimate compliance requirement and entirely achievable with the right architecture.

> First — clarify: 'production data' means the actual database records and disk blocks. The AWS infrastructure metadata (API calls, resource configurations) is a separate concern.

> Architecture for EU data residency:

> All AWS resources go into EU regions only — specifically `eu-central-1` (Frankfurt) or `eu-west-1` (Ireland) depending on the client's GDPR data residency requirements. A region-restriction SCP at the Org root level denies all API calls outside approved EU regions — so even if someone misconfigures something, the SCP prevents data leaving the EU.

> For MGN: the MGN Replication Server is in the EU AWS region. The source server's agent sends encrypted block-level data (TLS 1.2+) directly to the EU region. The data never touches a US or non-EU AWS service. I can provide the MGN replication endpoint DNS and show it resolves to an EU IP range.

> For DMS: the replication instance is in the EU region. Same principle — data flows from on-prem (EU) to the EU AWS region.

> For the evidence package for auditors:
> 1. SCP screenshot showing region restriction to EU only
> 2. CloudTrail logs showing all resource creation events happened in `eu-central-1`
> 3. MGN replication server's EC2 instance showing `eu-central-1` in the metadata
> 4. Network flow diagram showing data path: on-prem (EU) → Direct Connect → AWS EU → EU RDS/EC2
> 5. TLS certificate evidence (MGN uses TLS, data encrypted in transit)
> 6. KMS key configuration showing keys are in the EU region (KMS keys are region-specific)

> I'd also recommend an AWS Well-Architected Review with a focus on the EU data residency requirements before the first production wave — this produces a formal report the auditors can review."

---

**Q23: SCENARIO — You join a new migration project in week 3. You're asked to review the existing Terraform code. You find the following: S3 bucket with `acl = "public-read"`, security group with `ingress { from_port=22, to_port=22, cidr_blocks=["0.0.0.0/0"] }`, and RDS instance with `publicly_accessible = true`. What do you do?**

> **What the interviewer is assessing:** Security posture recognition, ability to push back on bad patterns, prioritisation.

**Model Answer:**

> "These are three separate critical security findings and I'd address them urgently, not eventually.

> Immediate response: I'd raise this in a written message to the tech lead and project manager today — not verbally, because I want a paper trail. I'd frame it as: 'I've identified three high-severity security misconfigurations in the infrastructure code that need to be remediated before any production data is migrated. Here's what they are, why they're critical, and the fixes.'

> Finding 1 — Public S3 bucket: any file uploaded to this bucket is readable by the entire internet. If any production data, backups, or logs land here, it's an immediate data breach. Fix: remove `acl = "public-read"`, add `aws_s3_bucket_public_access_block` resource with all four properties set to `true`, and add an explicit `Deny` bucket policy for `s3:GetObject` to all principals except approved ones.

> Finding 2 — SSH open to 0.0.0.0/0: port 22 open to the internet means any IP in the world can attempt to authenticate. Even with key-based auth, this is a brute-force and credential-theft risk. Fix: remove the `0.0.0.0/0` ingress on port 22. Replace SSH access with AWS Systems Manager Session Manager — no port 22 needed at all, all sessions logged to CloudTrail, no key management. If SSH is truly needed, restrict to the company's VPN IP range only.

> Finding 3 — RDS publicly accessible: this puts the DB on a public IP, meaning it's reachable from the internet if the security group allows it. Even if the security group is correct today, one misconfiguration away from a direct DB connection from anywhere in the world. Fix: `publicly_accessible = false`, place RDS in private subnets, access only from the application tier security group.

> I'd also recommend we add AWS Config rules to detect these patterns going forward: `s3-bucket-public-read-prohibited`, `restricted-ssh`, `rds-instance-public-access-check`. And run `checkov` or `tfsec` as part of the CI pipeline on every Terraform PR to catch these before they merge."

---

**Q24: How would you estimate the effort for migrating 300 on-prem servers to AWS for a client? They want a number by end of week.**

> **What the interviewer is assessing:** Can they scope a migration project? Not just technical skills — consulting skills, effort estimation, assumptions.

**Model Answer:**

> "I'd push back slightly on 'by end of week' without discovery data, but I'd give a range based on industry benchmarks with clear assumptions, pending refinement from actual discovery.

> The estimation framework I'd use:

> First — collect what's available now: the server inventory (how many are Windows vs. Linux, what sizes, what applications), any existing documentation on dependencies, a rough split of how many are 'easy' (standalone apps) vs. 'complex' (multi-tier with databases, third-party integrations).

> For 300 servers, typical migration benchmarks from MAP experience:
> - Assessment and Mobilize phase: 8-12 weeks (does not depend on server count — it's about Landing Zone, runbooks, tooling)
> - Migration velocity (once tooling is ready): 20-40 servers per wave, roughly 1-2 waves per month
> - Post-cutover validation: 30 days per wave before decommission
> - For 300 servers: estimate 8-12 waves over 8-14 months for migration execution

> Effort in person-days (rough ranges):
> - 2 DevOps engineers full-time on migration (MGN, IaC, cutovers): primary resource throughout
> - 1 DBA part-time (30% of their time) for DB migrations
> - 1 Security engineer part-time (20%) for Landing Zone and compliance
> - 1 Project Manager throughout

> My estimate for end of week would be: '8-14 months elapsed time for full migration, 2-3 DevOps engineers, 1 part-time DBA, pending a 2-week discovery sprint to refine the wave plan and identify complex applications that need re-scoping.'

> I'd be clear about the assumptions baked into that estimate: network connectivity (Direct Connect) established by end of Mobilize, no major heterogeneous DB migrations (those add 4-8 weeks each), no SAP or mainframe systems in scope, and business stakeholders available for cutover approval within 24 hours of request."

---

## Section 6 — Behavioral Questions (STAR Format Expected)

Behavioral questions assess judgment, ownership, and professionalism. Use the STAR format: **Situation, Task, Action, Result**.

---

**Q25: Tell me about a time a migration cutover went wrong. What happened, what did you do, and what did you change afterward?**

> **What the interviewer is looking for:** Genuine failure story (candidates who claim no failures have no real experience), ability to stay calm, structured problem-solving, ownership of the outcome, learning and process improvement.

**Model Answer (template — adapt to your real experience):**

> "Situation: We were cutting over a Java application with a PostgreSQL database from on-prem to Aurora on a Saturday night. DMS replication had been running clean for 2 weeks.

> Task: I was the lead engineer running the cutover — I owned the runbook and the go/no-go decision.

> Action: At 1AM, after we stopped DMS and validated row counts, we updated the application connection string and restarted the app. It came up, smoke tests passed — but about 8 minutes in, we started seeing 500 errors on a specific API endpoint. Digging into the logs, we found that a stored function used by that endpoint had a PostgreSQL syntax that was slightly different from the SCT conversion — it worked in testing but failed under the specific data pattern that only existed in production. We were now past our planned cutover window.

> I made the call to roll back: reverted the connection string to on-prem PostgreSQL, restarted the app on the original servers, and confirmed everything was working within 20 minutes. We had 25 minutes of degraded service before the rollback completed.

> Result: No data was lost because DMS CDC was still running — the on-prem database was fully up to date. The application was back on the original system. The failure was documented as a P2 incident.

> What we changed: we added a specific test for that stored function with production-like data patterns into the smoke test suite. We also added a 'canary endpoint test' step to the pre-cutover checklist that hits every major API endpoint once before we consider the cutover successful. The next attempt two weeks later was clean."

---

**Q26: Describe a situation where you had to push back on a technical decision made by a more senior person. How did you handle it?**

> **What the interviewer is looking for:** Technical confidence, professional assertiveness, outcome-focus.

**Model Answer (template):**

> "Situation: During a migration project, the solution architect recommended we use a single AWS account for all environments — dev, staging, and production — separated by VPCs. Their reasoning was it would simplify cross-environment connectivity.

> Task: I had specific concerns about the blast radius and compliance implications, and I needed to make my case clearly without making it personal.

> Action: I prepared a short written comparison — single account vs. multi-account — specifically addressing the trade-offs relevant to this client: blast radius if a developer's access key was compromised in dev (could access prod in the same account), the inability to apply different SCPs to dev vs. prod, and the complexity of cross-account billing breakdown for the finance team. I also included the AWS Well-Architected Framework guidance that explicitly recommends account-per-environment for production workloads. I scheduled a 30-minute call, presented the comparison, and specifically asked: 'Is there a constraint I'm not aware of that makes single-account preferable?'

> Result: The architect acknowledged the blast radius concern was valid and that they hadn't considered the SCP limitation. We agreed on a compromise — separate accounts for production and non-production, and within non-production we'd use VPC separation. This was actually simpler than full account-per-environment while addressing the critical concerns. The architect thanked me for pushing back with evidence rather than just objecting."

---

**Q27: You're in the middle of a migration and the client's team keeps requesting scope changes — 'can we also migrate this server?' and 'can you also set up monitoring for X?' How do you handle this?**

> **What the interviewer is looking for:** Consulting maturity, project management awareness, client communication, saying no professionally.

**Model Answer:**

> "This is a very common dynamic on migration projects and it's actually one of the harder soft-skill challenges — technical changes are often easier than managing scope.

> My approach: I'd acknowledge the request positively — the client is engaged, which is good. Then I'd make the scope impact explicit: 'Adding this server to Wave 3 would push the wave from 15 to 16 servers, which is manageable — but if we're adding 10 more across the wave, we should talk about timeline.' I'd log every request in a scope change register (a simple spreadsheet or Jira board), even small ones, so there's a visible record.

> For monitoring requests: I'd distinguish between migration-critical monitoring (which is in scope — we need to validate that migrated apps are healthy) and operational monitoring that goes beyond the migration brief. For the latter, I'd say: 'This is a great idea and we should capture it. Let's add it to a post-migration optimisation backlog so it doesn't delay the current wave.'

> The key principle: I'm on the client's side, but I'm also responsible for delivering the project on time. If scope grows unchecked, the DC lease date becomes a problem and the client is the one who suffers. Framing it as 'protecting the deadline' rather than 'refusing to help' is how I'd communicate it."

---

## Section 7 — Red Flags and Green Flags

### Red Flags — These Disqualify or Significantly Downgrade a Candidate

| Red Flag | What It Signals |
|---|---|
| Says "we used Terraform" but can't write a resource block | Watched someone else use it, no personal hands-on |
| Claims to have run migrations but doesn't know what MGN/DMS does | Resume inflation |
| Can't name the 7 Rs | Basic migration literacy gap |
| Describes security as "the security team's job" | Not DevSecOps mindset |
| Has never had a cutover fail or doesn't have a rollback story | Either inexperienced or dishonest |
| Can't explain IAM trust policies | Critical gap for any AWS migration role |
| Terraform state is "just a file" — doesn't know remote backends | Team Terraform experience is shallow |
| Says "just use the console" for repeatable tasks | Not infrastructure-as-code minded |
| Can't explain the difference between Multi-AZ and Read Replica | Basic RDS gap |
| Gives vague answers: "it depends" without explaining on what | Avoidance; can't make decisions |
| Gets defensive when asked about past failures | Ego risk on client-facing projects |

### Green Flags — These Elevate a Candidate

| Green Flag | What It Signals |
|---|---|
| Gives specific numbers: "80 servers", "6-week timeline", "3TB database" | Real experience, not theory |
| Mentions lessons learned from failures without prompting | Self-awareness, growth mindset |
| Asks clarifying questions before answering scenario questions | Engineering judgment |
| Knows the limits of a service (e.g., "MGN doesn't work for X") | Deep hands-on, not just marketing |
| Mentions cost implications alongside technical decisions | FinOps awareness — rare in junior engineers |
| Has used Infrastructure as Code for the Landing Zone, not just apps | Platform engineering maturity |
| Knows newer features: EKS Pod Identity (2024), Karpenter, EKS Auto Mode | Keeps current |
| Can explain trade-offs without needing to be asked | Consultative thinking |
| Has communicated architecture decisions to non-technical stakeholders | Client-facing readiness |
| Mentions running `checkov` or `tfsec` in CI pipelines | Security-first IaC |

---

## Section 8 — Quick-Fire Technical Checklist Questions

These are 1-2 minute yes/no depth checks the interviewer uses to quickly probe breadth:

| # | Question | What a Good Answer Includes |
|---|---|---|
| 1 | What is `terraform import` used for? | Bringing manually-created resources under Terraform management; syntax: `terraform import resource_type.name resource_id` |
| 2 | What is the difference between `terraform plan` and `terraform apply`? | Plan = dry run, shows what will change; apply = executes the changes; always review plan before apply |
| 3 | How do you roll back a bad Terraform apply? | `terraform apply` previous state version, or if resource creation: `terraform destroy -target`, or revert the code and `apply` again |
| 4 | What is a Kubernetes ConfigMap? | Key-value store for non-sensitive configuration; mounted as env vars or files into pods; not for secrets |
| 5 | What is the difference between a Kubernetes `Deployment` and a `StatefulSet`? | Deployment: stateless, pods interchangeable. StatefulSet: stable identity (hostname), stable storage, ordered operations — used for DBs, Kafka, Zookeeper |
| 6 | What is `kube-proxy`? | DaemonSet on every node that maintains iptables/IPVS rules for Service-to-Pod routing |
| 7 | How does an ALB Ingress work in EKS? | AWS Load Balancer Controller watches Ingress resources and provisions ALBs. Annotations on the Ingress configure the ALB (subnets, certificate, WAF, target type) |
| 8 | What is VPC CNI in EKS? | AWS's CNI plugin that assigns pod IPs directly from the VPC subnet CIDR — pods are first-class VPC citizens, routable from on-prem without extra encapsulation |
| 9 | What is AWS Secrets Manager vs. SSM Parameter Store? | Both store secrets. Secrets Manager: auto-rotation, native RDS/EKS integration, higher cost. SSM Parameter Store: cheaper, SecureString encrypted with KMS, no auto-rotation |
| 10 | What is IMDSv2 and why should you enforce it? | v2 requires a PUT request to get a session token before reading metadata — prevents SSRF attacks stealing IAM credentials. Enforce via `HttpTokens=required` on launch template |
| 11 | How do you prevent S3 data exfiltration in a migration? | S3 bucket policies with explicit Deny for `s3:PutObject` from outside org, Block Public Access, VPC endpoints for private S3 access, SCPs restricting S3 access to approved accounts |
| 12 | What is AWS Config's purpose? | Continuous compliance monitoring — records all resource configuration changes, evaluates against rules (managed or custom), reports non-compliant resources |
| 13 | What is GuardDuty and what does it detect? | ML-based threat detection analyzing CloudTrail, VPC Flow Logs, DNS logs, and EKS audit logs. Detects: credential abuse, crypto mining, unusual API calls, port scans, malware |
| 14 | What is the difference between `apply` and `push` in Argo CD? | `apply` is standard kubectl; Argo CD uses `sync` to reconcile the desired Git state with the live cluster state |
| 15 | What is a Helm chart and when would you use it over raw Kubernetes manifests? | Helm packages K8s manifests as templates with variable substitution. Use for complex applications with multiple resources that need environment-specific values |

---

## Section 9 — Scoring Rubric

The interviewer uses this after all rounds to make a hire/no-hire decision:

| Category | Weight | Score 1 (Weak) | Score 3 (Meets Bar) | Score 5 (Strong) |
|---|---|---|---|---|
| **AWS Migration Knowledge** | 25% | Knows 7 Rs conceptually, never used MGN/DMS | Has used MGN or DMS on a real project, can explain the flow | Has run cutovers, debugged replication issues, designed wave plans |
| **Terraform/IaC** | 20% | Copy-pastes examples, doesn't understand modules | Writes modules, manages remote state, CI pipeline | Designs module structure for multi-account, handles state issues, drift detection |
| **Containers (EKS/ECS)** | 20% | Has deployed a Docker container, knows what Kubernetes is | Has deployed to EKS/ECS in production, understands IRSA, Helm | Migrated K8s clusters, designed Karpenter/autoscaling, handles storage migration |
| **DevSecOps / Security** | 15% | Security is "not my job", doesn't know basic IAM | Knows IAM, SCPs, GuardDuty, encrypts everything by default | Designs Landing Zone security baseline, writes compliance-as-code, runs tfsec in CI |
| **Problem-Solving / Scenarios** | 15% | Gives generic answers, doesn't ask clarifying questions | Structured approach, identifies the key decision points | Considers trade-offs, mentions rollback, communicates to stakeholders |
| **Client/Communication** | 5% | Only technical, can't explain to non-engineers | Can explain trade-offs simply, handles scope change professionally | Has done client-facing work, proactively manages expectations |

**Hire threshold:** Average ≥ 3.5 across all categories with no category below 2.

---

## Section 10 — What to Say in Your 30-Second Summary at the End

Interviewers often close with: *"Is there anything you'd like us to know that we didn't cover?"*

This is your moment. Don't waste it with "No, I think we covered everything."

**Model Closing:**

> "I'd just reinforce that what I find genuinely interesting about migration work is that it combines infrastructure engineering, architecture design, and real project delivery under deadline pressure — and those three things don't often come together in pure greenfield work.

> One thing I didn't get to mention: I've been working on building out a personal lab for migration scenarios — I've set up an on-prem-equivalent with VirtualBox VMs, installed MGN agents, and run test migrations to a personal AWS account to keep the muscle memory sharp between projects. I also have a private GitHub repository with Terraform modules for Landing Zone components that I've been refining based on the patterns I've seen work and not work in real engagements.

> I'm genuinely excited about the prospect of working on client migration projects — the combination of technical depth and client collaboration is exactly the kind of environment I do my best work in."

---

## Quick Reference: Things to Memorise Before This Interview

```
7 Rs:           Retire, Retain, Rehost, Relocate, Replatform, Repurchase, Refactor
MAP Phases:     Assess → Mobilize → Migrate & Modernize
MGN vs DMS:     MGN = whole servers to EC2 | DMS = database data between engines
Multi-AZ vs RR: Multi-AZ = sync/HA/not readable | Read Replica = async/scaling/readable
IRSA flow:      OIDC provider → JWT token in pod → STS AssumeRoleWithWebIdentity → temp creds
SCP vs IAM:     SCP = account ceiling (can't be overridden) | IAM = principal permissions
Landing Zone:   Mgmt + Log Archive + Security + Network + Shared Services + Workloads accounts
Direct Connect: Private, dedicated, 1-100Gbps | VPN = IPSec over internet, cheaper, faster to set up
TGW:            Hub-and-spoke for VPCs; replaces VPC peering mesh at scale
Cutover:        Maintenance mode → wait for DMS lag=0 → validate → DNS switch → smoke tests → monitor
IMDSv2:         HttpTokens=required → prevents SSRF credential theft
Canary:         5% → 25% → 100% traffic shift with metric gates
Strangler Fig:  Proxy in front of monolith → extract one service at a time → route by path
```

---

*Document purpose: Interview preparation from the hiring manager's lens. Use every question as a practice prompt — answer it out loud before reading the model answer.*

*Related files in this series:*
- `AWS Migration-Intro.md` — Full technical reference for migration concepts
- *(Coming next)* `AWS Migration - Patterns.md` — Deep-dive runbooks per migration type
