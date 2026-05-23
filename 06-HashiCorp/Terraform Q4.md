# Terraform Interview Question 4 — Hung Apply & State Backup/Restore

> **Topic:** Diagnosing a hung `terraform apply`, safely backing up and restoring Terraform state, and architectural patterns that prevent state corruption
> **Level:** Intermediate–Advanced
> **Relevance:** A "terraform apply never finishes" is one of the most stressful real-world incidents — interviewers use it to test whether you stay methodical under pressure and know how to protect state without data loss

---

## ❓ The Question

> **"Your `terraform apply` is running but seems stuck — it's been 15 minutes with no progress. Walk me through exactly what you do."**

Common follow-ups:
- *"How do you back up Terraform state before taking action?"*
- *"What does `terraform state pull` actually do, and how is it different from just copying the state file?"*
- *"If the state file gets corrupted after you Ctrl+C, how do you recover?"*
- *"What is a tainted resource and how does it relate to a failed apply?"*
- *"How would you architect state to minimize the blast radius if this happens?"*

---

## 🧠 Why terraform apply Hangs — Full Root Cause Map

Before touching anything, understand *why* it's hanging. There are five distinct categories:

### 1. Provider API Throttling or Timeout
The cloud provider (AWS, GCP, Azure) is rate-limiting your requests or the resource creation itself is taking longer than expected. Terraform is waiting for the API to respond — it hasn't crashed.

```
terraform apply
  ↓ Creates aws_instance.web
  ↓ Waits for instance to reach "running" state
  ↓ ... [15 minutes later, instance still "pending"] ...  ← hung here
```

Common trigger: AWS API throttling during large batch applies, or RDS instance creation taking 20+ minutes.

### 2. State Lock Held by Another Process
A previous `terraform apply` or plan acquired a DynamoDB lock and never released it. Your current apply is waiting for the lock to be freed — it will wait indefinitely by default.

```bash
# What you'd see in the terminal:
# Acquiring state lock. This may take a few moments...
# [stuck here — lock never acquired]
```

### 3. Network Connectivity Lost Mid-Apply
The connection from your CI runner or laptop to the cloud provider API was interrupted after some resources were created but before others. Terraform is retrying silently in the background.

### 4. Resource Dependency Deadlock
A circular or unresolved dependency between resources causes Terraform to wait for a resource that can never finish. Rare with proper HCL but possible with `depends_on` misuse.

### 5. Provisioner Hung (remote-exec / local-exec)
An SSH session opened by `remote-exec` is waiting for a command that never finishes (an install script that blocks, a service that doesn't start). This is the most common cause of multi-hour hangs.

```hcl
# Classic hang: waiting for package download with no timeout
provisioner "remote-exec" {
  inline = ["sudo apt-get install -y some-package"]
  # apt-get is waiting for a locked dpkg or a slow mirror
  # No timeout = Terraform waits forever
}
```

---

## 🔍 Step-by-Step Debugging Workflow

### Step 1 — Don't Touch Terminal One Yet

Resist the urge to Ctrl+C immediately. First, open **Terminal Two** and gather information.

```bash
# What resources did Terraform successfully create?
# Check the lock — is someone else holding it?
aws dynamodb get-item \
  --table-name terraform-lock \
  --key '{"LockID": {"S": "my-bucket/env/prod/terraform.tfstate-md5"}}' \
  --region us-east-1

# Sample output when lock is held:
# {
#   "Item": {
#     "Info": {"S": "{\"ID\":\"abc123\",\"Operation\":\"OperationTypeApply\",
#                    \"Who\":\"runner@github-actions\",\"Created\":\"2024-01-15T10:00:00Z\"}"}
#   }
# }
```

```bash
# Check if the original process is actually still alive (on local machine)
ps aux | grep terraform

# Check cloud-side: are resources still being created?
aws ec2 describe-instances --filters "Name=tag:ManagedBy,Values=terraform" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name]' \
  --output table
```

### Step 2 — Back Up State Immediately (Before Anything Else)

```bash
# ALWAYS do this before any recovery action
# terraform state pull fetches the latest remote state to stdout
terraform state pull > terraform.tfstate.backup.$(date +%Y%m%d-%H%M%S)

# Verify the backup is valid JSON and not empty
wc -c terraform.tfstate.backup.*
python3 -c "import json,sys; json.load(open(sys.argv[1])); print('Valid JSON')" \
  terraform.tfstate.backup.*

# For S3 backend: also snapshot directly from S3 (belt and suspenders)
aws s3 cp s3://my-terraform-state/prod/terraform.tfstate \
  ./terraform.tfstate.s3-backup.$(date +%Y%m%d-%H%M%S)
```

**Why `terraform state pull` instead of just copying the file?**

| Approach | What It Does | When to Use |
|----------|-------------|-------------|
| `terraform state pull` | Fetches the *current locked state* from the backend (S3, GCS, Consul) via Terraform's own code | Always — this is the authoritative copy |
| `cp terraform.tfstate` | Copies a local file — only relevant for local backends | Local backend only |
| `aws s3 cp` | Copies from S3 directly, bypassing Terraform — good as a secondary backup | Additional safety snapshot |

`terraform state pull` respects the backend configuration and always retrieves the current version — it won't give you a stale local copy.

### Step 3 — Cancel the Hung Process

Once you have a backup, Ctrl+C in Terminal One.

```
^C
╷
│ Warning: Apply cancelled
│
│ Terraform will attempt to return the state to a consistent state. You may need
│ to manually run `terraform apply` again to complete the apply.
╵
```

**What happens to the lock when you Ctrl+C?**

Terraform catches the interrupt signal and attempts to release the DynamoDB lock before exiting. In most cases it succeeds. However, if the process is killed with SIGKILL (`kill -9`) or the CI runner is terminated abruptly, the lock remains held. Then you need `terraform force-unlock` (covered in Terraform Q3).

### Step 4 — Assess State Integrity

```bash
# Check if state is valid after the interrupted apply
terraform state list

# If this errors with "state file invalid" or "unexpected EOF":
# → State is corrupted, proceed to recovery section below

# Check which resources might be in a partially-created state
terraform plan
# Look for: "1 to add, 0 to change, 1 to destroy" — the partially-created resource
# may show up as a tainted resource
```

### Step 5 — Handle Tainted Resources

When Terraform's `apply` is interrupted mid-creation, Terraform marks the partially-created resource as **tainted** — meaning it will be destroyed and re-created on the next apply.

```bash
# List all tainted resources
terraform state list | xargs -I{} terraform state show {}
# Tainted resources show: status = "tainted"

# Verify in state file:
terraform state show aws_instance.web
# If partially created, it shows as tainted

# Option 1: Let Terraform handle it (recommended)
terraform apply
# Terraform will: destroy tainted resource → re-create it

# Option 2: Manually untaint if you know the resource is actually fine
terraform untaint aws_instance.web
# Then apply — Terraform will try to update rather than replace

# Option 3: Remove from state entirely if resource was never created
terraform state rm aws_instance.web
# Then apply — Terraform creates it fresh
```

### Step 6 — Re-Run Apply

```bash
terraform apply
# If clean: apply completes normally
# If errors: see state recovery below
```

---

## 🔧 State Recovery — When State Is Corrupted

### Detect Corruption

```bash
# Attempt to list resources
terraform state list
# Error: Failed to load state: unexpected end of JSON input
# → State file is truncated/corrupted

# Attempt to validate JSON directly
python3 -c "import json; json.load(open('terraform.tfstate'))"
# SyntaxError: Expecting property name → JSON is malformed
```

### Restore from Backup

```bash
# Method 1: Push your backup back as the current state
# terraform state push is the inverse of terraform state pull
terraform state push terraform.tfstate.backup.20240115-100523

# Terraform will prompt: "Do you want to override the remote state?"
# Answer yes only if you've verified the backup is the correct state

# Verify push was successful
terraform state list
terraform plan   # Should show minimal drift
```

**`terraform state push` safety checks:**

- Terraform checks the **serial number** in the state file. If your backup has a *lower* serial than the current remote state (meaning someone else has made changes), the push will be rejected unless you pass `-force`
- **Never use `-force` without understanding what you're overwriting**

```bash
# Check serials before pushing
# Your backup:
cat terraform.tfstate.backup.* | python3 -c "import json,sys; s=json.load(sys.stdin); print('Serial:', s['serial'])"

# Current remote state:
terraform state pull | python3 -c "import json,sys; s=json.load(sys.stdin); print('Serial:', s['serial'])"

# Only push if backup serial >= current serial
```

### Recover from S3 Versioning

If you use S3 with versioning enabled (the correct setup), you can recover any previous version:

```bash
# List all versions of the state file
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified,IsLatest]' \
  --output table

# Output:
# VersionId          | LastModified              | IsLatest
# abc123xyz          | 2024-01-15T09:58:00.000Z  | True   ← corrupted
# def456uvw          | 2024-01-15T09:45:00.000Z  | False  ← last good state
# ghi789rst          | 2024-01-15T09:00:00.000Z  | False

# Restore the last known-good version
aws s3api get-object \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate \
  --version-id def456uvw \
  terraform.tfstate.recovered

# Push the recovered state
terraform state push terraform.tfstate.recovered
```

---

## 🏗️ Modular State Architecture — Prevention Over Recovery

The real answer to "terraform apply hangs" at scale is architectural: **don't have one giant state file**.

### The Problem with Monolithic State

```
# One state file for everything:
terraform apply   # Creates VPC, subnets, RDS, ECS cluster, ALB, Route53...
# If this hangs at resource #47 of 200:
# - Lock is held for entire codebase
# - Blast radius of a corrupted state = all 200 resources
# - Recovery time = hours
```

### Separated State by Layer

```
infrastructure/
├── 00-networking/          # VPC, subnets, internet gateway
│   ├── main.tf
│   └── terraform.tfstate   # → s3://bucket/networking/terraform.tfstate
├── 01-security/            # Security groups, IAM roles
│   └── terraform.tfstate   # → s3://bucket/security/terraform.tfstate
├── 02-data/                # RDS, ElastiCache, S3 buckets
│   └── terraform.tfstate   # → s3://bucket/data/terraform.tfstate
├── 03-compute/             # ECS cluster, ASG, EC2 instances
│   └── terraform.tfstate   # → s3://bucket/compute/terraform.tfstate
└── 04-application/         # ECS services, Route53, CloudFront
    └── terraform.tfstate   # → s3://bucket/application/terraform.tfstate
```

**Benefits:**
- A hung apply in `03-compute` doesn't lock `02-data`
- Corrupted state in one layer doesn't affect others
- Each layer has its own backup/recovery scope
- Faster applies — each workspace is small
- Teams can work on different layers simultaneously

### Cross-Layer Data Sharing with Remote State Data Sources

```hcl
# 03-compute/main.tf — reading outputs from the networking layer
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  # Use the VPC subnet ID from the networking layer's state
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  # ...
}
```

---

## 🔄 Terraform State Pull/Push Reference

```bash
# PULL — read remote state to stdout or file
terraform state pull                          # Print to stdout
terraform state pull > backup.tfstate         # Save to file
terraform state pull | jq '.resources[].type' # Inspect with jq

# PUSH — write a local file to the remote backend
terraform state push backup.tfstate           # Normal push (serial check)
terraform state push -force backup.tfstate    # Override serial check (DANGEROUS)

# STATE SERIAL — every write increments this counter
# Used to prevent out-of-order writes
terraform state pull | jq '.serial'
# → 42   (means 42 successful writes have been made to this state)
```

### Full Backup Script for CI/CD

```bash
#!/bin/bash
# terraform-state-backup.sh
# Run before any risky terraform operation

set -euo pipefail

BACKUP_DIR="./state-backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/terraform.tfstate.${TIMESTAMP}"

mkdir -p "${BACKUP_DIR}"

echo "[INFO] Pulling current state..."
terraform state pull > "${BACKUP_FILE}"

# Validate
FILE_SIZE=$(wc -c < "${BACKUP_FILE}")
if [ "${FILE_SIZE}" -lt 100 ]; then
  echo "[ERROR] State file suspiciously small (${FILE_SIZE} bytes) — aborting"
  exit 1
fi

python3 -c "import json,sys; json.load(open('${BACKUP_FILE}')); print('[INFO] State JSON valid')"

SERIAL=$(python3 -c "import json; print(json.load(open('${BACKUP_FILE}'))['serial'])")
echo "[INFO] State backup saved: ${BACKUP_FILE} (serial: ${SERIAL})"

# Also store in S3 with explicit versioned key
if [ -n "${BACKUP_S3_BUCKET:-}" ]; then
  aws s3 cp "${BACKUP_FILE}" \
    "s3://${BACKUP_S3_BUCKET}/state-backups/terraform.tfstate.${TIMESTAMP}"
  echo "[INFO] Backup also stored in S3"
fi
```

---

## ⚙️ Timeout Configuration — Preventing Hangs at the Source

Many hangs can be prevented by setting explicit timeouts in your Terraform configuration.

### Resource-Level Timeouts

```hcl
resource "aws_db_instance" "main" {
  # ...

  timeouts {
    create = "40m"   # Default: 40m — adjust for slow RDS creation
    update = "80m"   # Default: 80m — major version upgrades take time
    delete = "60m"   # Default: 60m
  }
}

resource "aws_eks_cluster" "main" {
  # ...
  timeouts {
    create = "30m"   # EKS cluster creation can take 10-20 minutes
    update = "60m"
    delete = "15m"
  }
}

resource "aws_elasticache_replication_group" "redis" {
  # ...
  timeouts {
    create = "60m"
    update = "40m"
    delete = "40m"
  }
}
```

### Provider-Level Timeouts

```hcl
provider "aws" {
  region = "us-east-1"

  # Retry failed API calls up to 10 times before failing
  max_retries = 10

  # HTTP client timeout per request
  # Prevents infinite wait on API hangs
  http_proxy = ""
}
```

### Provisioner Timeouts

```hcl
connection {
  type    = "ssh"
  timeout = "5m"   # Give up SSH if instance not ready in 5 minutes
  host    = self.public_ip
}

provisioner "remote-exec" {
  inline = [
    # Always add timeouts inside provisioner scripts too
    "timeout 300 sudo apt-get install -y nginx || exit 1",
  ]
}
```

---

## 🚨 Real-World Scenario — CI Pipeline Hung Apply Recovery

**Incident:** A GitHub Actions workflow running `terraform apply` for a production ECS service was stuck at "Creating aws_ecs_service.api" for 40 minutes. The job would hit GitHub's 6-hour job timeout if left unattended, leaving the lock held and blocking all other pipeline runs.

**Root cause:** The new ECS task definition referenced a container image that had been accidentally deleted from ECR. ECS kept retrying to pull the image — the service never reached `RUNNING` state — so Terraform waited for the service to stabilize.

**Recovery:**

```bash
# Step 1 (Terminal 2 on local machine): Back up state
terraform state pull > state-backup-incident-$(date +%Y%m%d-%H%M%S).json

# Step 2: Check ECS service events to confirm the cause
aws ecs describe-services \
  --cluster prod-cluster \
  --services api-service \
  --query 'services[0].events[0:5]'
# → "CannotPullContainerError: pull access denied for ecr.../api:v2.1.4, 
#    repository does not exist"

# Step 3: Fix the root cause BEFORE canceling Terraform
# Push the missing image back to ECR
docker pull nginx:latest   # placeholder
docker tag nginx:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/api:v2.1.4
aws ecr get-login-password | docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/api:v2.1.4

# Step 4: Cancel the GitHub Actions job (via UI or API)
# The lock was released cleanly because GitHub sends SIGTERM → Terraform catches it

# Step 5: Verify lock is released
aws dynamodb get-item \
  --table-name terraform-lock \
  --key '{"LockID": {"S": "prod-state-bucket/ecs/terraform.tfstate-md5"}}'
# → {} (empty = no lock)

# Step 6: Re-run the pipeline — now succeeds because image exists
```

**Prevention added after incident:**
- Added ECR image existence check as a pre-step in the pipeline
- Added `timeouts { create = "10m" }` to all `aws_ecs_service` resources so future hangs fail fast instead of waiting indefinitely
- Enabled ECS service circuit breaker (`deployment_circuit_breaker { enable = true, rollback = true }`) so ECS rolls back failed deployments automatically

```hcl
resource "aws_ecs_service" "api" {
  # ...

  deployment_circuit_breaker {
    enable   = true    # Detect failed deployments
    rollback = true    # Auto-rollback to previous task definition
  }

  timeouts {
    create = "10m"   # Fail fast instead of waiting 40 minutes
    update = "10m"
    delete = "10m"
  }
}
```

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **Ctrl+C doesn't release lock** | If the Terraform process is killed (SIGKILL) or CI job is force-terminated, DynamoDB lock remains held | Check lock table after every failed run; add `terraform force-unlock` step to CI cleanup |
| **`state push` with lower serial** | Restoring a backup with serial 40 when remote state is at serial 45 gets rejected | Always pull and compare serials before pushing; use S3 versioning to find the right version |
| **State backup is empty** | `terraform state pull` returns `{}` — no resources | Check backend config is correct; may be authenticating to wrong environment |
| **Tainted resource re-creates on next apply** | Interrupted apply leaves resource tainted; next apply destroys and recreates it | Review with `terraform state show` first; untaint if resource is healthy |
| **Multiple engineers both hit Ctrl+C** | Both releases the lock, then both try to re-apply — first one grabs lock, second waits | Use CI/CD for production applies; never have two engineers applying the same workspace |
| **`on_failure = continue` hides provisioner hangs** | Provisioner hangs but apply keeps "running" due to `on_failure = continue` | Remove `on_failure = continue` from provisioners that can block; add explicit script timeouts |
| **S3 versioning not enabled** | State corruption with no version history → no recovery point | Enable `versioning { enabled = true }` on state bucket from day one |

---

## ✅ Best Practices

- **Always back up state before any recovery action** — `terraform state pull > backup-$(date +%Y%m%d-%H%M%S).json` takes 3 seconds and saves hours
- **Enable S3 versioning on your state bucket** — every `terraform apply` creates a new version; corrupted state is always one `get-object --version-id` away from recovery
- **Set explicit timeouts on slow resources** (RDS, EKS, ECS, ElastiCache) — default timeouts are long but not infinite; tune them to fail fast and alert early
- **Add ECS deployment circuit breakers** — `deployment_circuit_breaker { enable = true, rollback = true }` prevents ECS service creation from hanging indefinitely on bad task definitions
- **Use modular state** — split by layer (networking, data, compute, application); a hang in one layer doesn't block the others
- **Add a CI pipeline lock check step** — before `terraform apply`, check the DynamoDB table; if a lock exists and its `Created` timestamp is > 30 minutes old, alert and investigate before proceeding
- **Never `terraform force-unlock` without confirming the original process is dead** — if two applies run simultaneously against the same state, state corruption is almost guaranteed
- **Add `set -euo pipefail` to all provisioner scripts** — prevents silent hangs where a script command fails and the provisioner just waits at the next command

---

## 📋 Quick Reference

```bash
# Back up state (always first)
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).json

# Check for stuck lock
aws dynamodb get-item --table-name terraform-lock \
  --key '{"LockID": {"S": "BUCKET/KEY-md5"}}'

# Interrupt the hung apply
Ctrl+C

# Verify state after interrupt
terraform state list
terraform plan

# Handle tainted resource
terraform state show <resource>        # Check if tainted
terraform untaint <resource>           # Un-taint if resource is healthy
terraform apply                        # Or let apply destroy/recreate

# Restore state from backup
terraform state pull | jq '.serial'    # Check current serial
cat backup.json | jq '.serial'         # Check backup serial
terraform state push backup.json       # Push if backup serial >= current

# Recover from S3 versioning
aws s3api list-object-versions --bucket my-state-bucket --prefix prod/terraform.tfstate
aws s3api get-object --bucket my-state-bucket --key prod/terraform.tfstate \
  --version-id <VERSION_ID> recovered.tfstate
terraform state push recovered.tfstate

# Fail-fast: add timeouts to slow resources
resource "aws_ecs_service" "app" {
  timeouts { create = "10m"; update = "10m" }
  deployment_circuit_breaker { enable = true; rollback = true }
}
```

---

> **Interview Takeaway:** A hung `terraform apply` is a multi-step recovery problem, not a "just Ctrl+C" situation. The correct order is: (1) open a second terminal, (2) `terraform state pull > backup.json` before touching anything, (3) understand *why* it's hanging (lock? provisioner? API throttle?), (4) cancel the apply, (5) assess state integrity with `terraform state list` and `terraform plan`, (6) handle any tainted resources before re-applying. For prevention, the three high-value controls are: explicit resource timeouts, S3 versioning on the state bucket, and modular state design so one hung apply doesn't hold a lock across your entire infrastructure.
