# Terraform Interview Question 3 — State Lock & State Management

> **Topic:** Terraform state locking, remote backends, state commands, and state file security
> **Level:** Intermediate
> **Relevance:** State management is central to every real-world Terraform workflow — interviewers use this to test whether you've worked with Terraform in a team environment, not just solo on a laptop

---

## ❓ The Question

> **"What is Terraform state locking? Why does it matter? What happens if a lock gets stuck?"**

Follow-ups you'll commonly hear:
- *"What backends support state locking? How do you configure S3 + DynamoDB locking?"*
- *"What is `terraform force-unlock` and when would you use it — carefully?"*
- *"Walk me through the Terraform state commands: `list`, `show`, `mv`, `rm`."*
- *"What sensitive information ends up in the Terraform state file? How do you protect it?"*

---

## 🧠 Why State Locking Exists

Terraform tracks everything it manages in a **state file** — a JSON document that maps your Terraform resources to real infrastructure. Every `terraform apply` reads the state, computes a diff, makes API calls, and writes back the updated state.

Without locking, two simultaneous applies are catastrophic:

```
Time →
User A: reads state → plans changes → applies → writes state ──┐
User B: reads state → plans changes → applies → writes state ──┘
                                                               ↑
                               Last writer wins — the other's changes are silently overwritten
                               Both may try to create the same resource → duplicate or conflict
                               State now represents neither A nor B's full intent
```

With locking, it's sequential and safe:

```
User A: acquires lock → reads state → applies → writes state → releases lock
User B: tries to acquire lock → BLOCKED → waits → lock released → acquires lock → proceeds
```

---

## 🖼️ Reference Diagrams (Source: KodeKloud)

![Terraform state locking — automatic operation, prevention of corruption, -lock flag](https://kodekloud.com/kk-media/image/upload/v1752873359/notes-assets/images/DevOps-Interview-Preparation-Course-Terraform-Question-3/terraform-state-locking-feature.jpg)

![Terraform state locking diagram — User 1 applies, User 2 waits, sequential integrity](https://kodekloud.com/kk-media/image/upload/v1752873360/notes-assets/images/DevOps-Interview-Preparation-Course-Terraform-Question-3/terraform-state-locking-diagram.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **State file** | `terraform.tfstate` — JSON file mapping Terraform resources to real infrastructure |
| **State lock** | A lock acquired automatically by Terraform during any write operation |
| **Backend** | Where Terraform stores state (local, S3, GCS, Azure Blob, Terraform Cloud) |
| **DynamoDB lock table** | AWS DynamoDB table used by the S3 backend for state locking |
| **Lock ID** | Unique identifier for the current lock — needed for `force-unlock` |
| **`force-unlock`** | Command to manually release a stuck lock — use with extreme caution |
| **`-lock=false`** | Flag to disable locking for a single operation — strongly discouraged |
| **State drift** | Real infrastructure diverges from Terraform state due to manual changes |
| **`terraform refresh`** | Syncs state to match real infrastructure (deprecated — use `terraform apply -refresh-only`) |

---

## ⚙️ How State Locking Works — The Mechanics

### When locking happens

```bash
terraform plan    # Acquires a read lock (brief)
terraform apply   # Acquires a write lock for the entire apply duration
terraform destroy # Acquires a write lock
terraform import  # Acquires a write lock
```

`terraform validate` and `terraform fmt` do NOT touch state — no lock needed.

### What the lock looks like (DynamoDB)

When the S3 backend acquires a lock, it writes a record to DynamoDB:

```json
{
  "LockID":   "my-terraform-state/prod/terraform.tfstate-md5",
  "Info": {
    "ID":        "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "Operation": "OperationTypeApply",
    "Who":       "jenkins@ci-runner-01",
    "Version":   "1.7.0",
    "Created":   "2025-01-15T14:23:01.123456789Z",
    "Path":      "my-terraform-state/prod/terraform.tfstate"
  }
}
```

This tells you: who is running, what operation, and when it started. Invaluable when debugging a stuck lock.

---

## 🔧 Backend Configuration — S3 + DynamoDB (Most Common AWS Setup)

```hcl
# backend.tf

terraform {
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "services/user-api/terraform.tfstate"
    region         = "us-east-1"

    # Encryption at rest
    encrypt    = true
    kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abc-def-ghi"

    # State locking via DynamoDB
    dynamodb_table = "terraform-state-lock"

    # Optional: separate role for state access
    role_arn = "arn:aws:iam::123456789:role/terraform-state-role"
  }
}
```

### Create the DynamoDB lock table (Terraform itself)

```hcl
# bootstrap/main.tf — create state infrastructure before using it

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-prod"

  lifecycle {
    prevent_destroy = true   # Never accidentally delete the state bucket
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"   # Versioning = history of all state changes
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"   # No capacity planning needed
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "Terraform State Lock Table"
    ManagedBy = "terraform-bootstrap"
  }
}
```

### Other Backend Options

```hcl
# Google Cloud Storage (GCS) — built-in locking via GCS object lock
backend "gcs" {
  bucket = "my-terraform-state"
  prefix = "prod/user-api"
}

# Azure Blob Storage — built-in locking via blob lease
backend "azurerm" {
  resource_group_name  = "terraform-state-rg"
  storage_account_name = "myterraformstate"
  container_name       = "tfstate"
  key                  = "prod/user-api/terraform.tfstate"
}

# Terraform Cloud — built-in locking + remote execution
backend "remote" {
  organization = "my-org"
  workspaces {
    name = "user-api-prod"
  }
}
```

| Backend | Locking Mechanism | Notes |
|---------|------------------|-------|
| **Local** | File-based lock (`.terraform.tfstate.lock.info`) | Single user only |
| **S3** | DynamoDB table (must create separately) | Most common AWS setup |
| **GCS** | Native GCS object lock | Built-in, no extra setup |
| **Azure Blob** | Blob lease mechanism | Built-in, no extra setup |
| **Terraform Cloud** | API-based lock | Built-in + run queue |

---

## 🚨 Handling a Stuck Lock

### When does a lock get stuck?

- A `terraform apply` was running and the CI runner was killed mid-run
- Network partition between Terraform and DynamoDB
- Terraform process crashed (OOM kill, timeout)
- Someone's laptop lost connectivity mid-apply

### Diagnosing the stuck lock

```bash
# Error you'll see when a lock is held:
terraform apply
# Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException
#
# Lock Info:
#   ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
#   Path:      my-terraform-state/prod/terraform.tfstate
#   Operation: OperationTypeApply
#   Who:       jenkins@ci-runner-01
#   Version:   1.7.0
#   Created:   2025-01-15 14:23:01.123456789 +0000 UTC
#   Info:

# Step 1: Confirm the lock in DynamoDB
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "my-terraform-state/prod/terraform.tfstate-md5"}}'

# Step 2: Is the original operation still running?
# Check CI pipeline logs — is job a1b2c3d4... still active?
# If YES: wait for it to complete
# If NO (job crashed/killed): safe to force-unlock

# Step 3: Check state file integrity first
aws s3 cp s3://my-terraform-state/prod/terraform.tfstate /tmp/state-backup.json
python3 -m json.tool /tmp/state-backup.json > /dev/null && echo "State JSON is valid"
```

### Force-Unlock — Use With Extreme Caution

```bash
# Only use force-unlock if you are CERTAIN no operation is running
# Using it while an apply is in progress will corrupt state

terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Interactive confirmation will be prompted:
# Do you really want to force-unlock?
# Terraform will remove the lock on the remote state.
# This will allow local Terraform commands to modify this state, even though it
# may be still be in use. Only 'yes' will be accepted to confirm.
# Enter a value: yes
```

**The force-unlock checklist (do all of these before running it):**
1. ✅ Confirm the CI job that held the lock is no longer running
2. ✅ Check the lock's `Created` timestamp — if it's hours old, likely safe
3. ✅ Download a backup of the state file before unlocking
4. ✅ Get a second engineer to verify before proceeding
5. ✅ After unlocking, run `terraform plan` to verify state is consistent

### Preventing Stuck Locks in CI

```yaml
# GitHub Actions — add a timeout to prevent locks from being held indefinitely
jobs:
  terraform:
    timeout-minutes: 30    # CI job killed after 30 min → releases lock
    steps:
      - name: Terraform Apply
        run: terraform apply -auto-approve
        timeout-minutes: 25   # Inner timeout gives time for cleanup

      # Cleanup step: always runs, even if apply fails
      - name: Release lock if stuck
        if: failure()
        run: |
          # If apply failed mid-run, state may still be locked
          # Wait a moment, then check and force-unlock if needed
          sleep 10
          terraform plan 2>&1 | grep -q "Error locking state" && \
            terraform force-unlock -force $(terraform plan 2>&1 | grep "ID:" | awk '{print $2}') || true
```

---

## 🔑 State File — What's Inside and Why It's Sensitive

```json
// terraform.tfstate (simplified)
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 42,
  "lineage": "a1b2c3d4-...",
  "outputs": {
    "db_endpoint": {
      "value": "mydb.abc123.us-east-1.rds.amazonaws.com",
      "type": "string"
    }
  },
  "resources": [
    {
      "type": "aws_db_instance",
      "name": "main",
      "instances": [{
        "attributes": {
          "id":       "mydb",
          "endpoint": "mydb.abc123.us-east-1.rds.amazonaws.com",
          "username": "admin",
          "password": "MyS3cr3tP@ssword!"    ← PLAINTEXT in state!
        }
      }]
    },
    {
      "type": "aws_iam_access_key",
      "name": "deploy_key",
      "instances": [{
        "attributes": {
          "id":     "AKIAIOSFODNN7EXAMPLE",
          "secret": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"  ← PLAINTEXT!
        }
      }]
    }
  ]
}
```

**State stores in plaintext:** RDS passwords, IAM access key secrets, private keys, API tokens — anything passed to a resource attribute. This is why state file security is non-negotiable.

### Protecting State

```hcl
# 1. Always use encrypted remote backend
backend "s3" {
  encrypt    = true
  kms_key_id = "arn:aws:kms:..."   # Customer-managed KMS key
}

# 2. S3 bucket policy — deny any access not from trusted roles
resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  policy = jsonencode({
    Statement = [
      {
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource  = ["${aws_s3_bucket.terraform_state.arn}/*"]
        Condition = {
          StringNotEquals = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::123456789:role/terraform-ci-role",
              "arn:aws:iam::123456789:role/terraform-admin-role",
            ]
          }
        }
      }
    ]
  })
}
```

```bash
# 3. Enable S3 versioning — recover from accidental state corruption
# Every state write creates a new version — you can restore any previous version:
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table

# Restore a previous version
aws s3api copy-object \
  --copy-source my-terraform-state/prod/terraform.tfstate?versionId=<VERSION_ID> \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate
```

---

## 🛠️ State Management Commands

These commands let you inspect and surgically modify state — without making API calls to the cloud.

### `terraform state list` — See Everything in State

```bash
terraform state list
# aws_instance.web
# aws_db_instance.main
# aws_security_group.web
# module.vpc.aws_vpc.main
# module.vpc.aws_subnet.public[0]
# module.vpc.aws_subnet.public[1]

# Filter by module
terraform state list module.vpc
```

### `terraform state show` — Details of One Resource

```bash
terraform state show aws_instance.web
# # aws_instance.web:
# resource "aws_instance" "web" {
#     ami                   = "ami-0c02fb55956c7d316"
#     instance_type         = "t3.medium"
#     id                    = "i-0abc123def456789"
#     private_ip            = "10.0.1.50"
#     public_ip             = "54.123.45.67"
#     ...
# }
```

### `terraform state mv` — Rename a Resource in State

Used when you rename a resource in your `.tf` files and don't want Terraform to destroy and recreate it:

```bash
# You renamed aws_instance.web → aws_instance.app_server in your code
# Without state mv: Terraform would destroy + recreate the instance
# With state mv: Terraform just updates the state key

terraform state mv aws_instance.web aws_instance.app_server

# Moving a resource into a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Moving between workspaces requires manual state pull/push
```

### `terraform state rm` — Remove a Resource from State

Use when you want Terraform to "forget" a resource without destroying it (e.g., you're handing management to another team or another Terraform config):

```bash
# Remove from state (the real resource is NOT deleted)
terraform state rm aws_instance.legacy_server

# Now Terraform treats it as if it never managed it
# The EC2 instance still exists in AWS

# Remove an entire module
terraform state rm module.old_vpc
```

### `terraform state pull` and `push` — Manual State Operations

```bash
# Download current state to stdout
terraform state pull > state-backup.json

# Inspect it
cat state-backup.json | jq '.resources[].type' | sort | uniq -c

# Push a modified state (use with extreme caution — bypasses all safety)
terraform state push state-modified.json
# ⚠️ This can corrupt state — only use in recovery scenarios
```

### `terraform import` — Bring Existing Resources Under Terraform Management

```bash
# You have an EC2 instance created manually — bring it into state
terraform import aws_instance.web i-0abc123def456789

# Import an S3 bucket
terraform import aws_s3_bucket.data my-existing-bucket

# Import a module resource
terraform import module.vpc.aws_vpc.main vpc-0abc123

# After import: run terraform plan to see if config matches reality
terraform plan   # Should show no changes if your .tf matches the real resource
```

### `terraform apply -refresh-only` — Sync State to Reality

```bash
# Someone manually changed a security group in the console
# This updates state to reflect the real world (without making changes)
terraform apply -refresh-only

# Equivalent to the old terraform refresh (now deprecated)
# Use this to detect drift before planning changes
```

---

## 🌍 Real-World Scenario: Two CI Pipelines Fight Over State

**Situation:** Two GitHub Actions workflows both trigger on push to `main` — one deploys infrastructure, one deploys the app. Both run `terraform apply`. They both start at the same time.

```
Pipeline A (infrastructure): terraform apply → acquires lock at 14:23:01
Pipeline B (app deploy):     terraform apply → ERROR: state is locked by Pipeline A
```

**Pipeline B error:**
```
Error: Error locking state: Error acquiring the state lock
Lock Info:
  ID:        a1b2c3d4...
  Who:       github-actions@runner-ubuntu-latest
  Operation: OperationTypeApply
  Created:   2025-01-15 14:23:01 UTC
```

**Fix — use GitHub Actions concurrency to serialize:**

```yaml
# Both workflows use the same concurrency group
concurrency:
  group: terraform-prod
  cancel-in-progress: false   # Don't cancel, queue instead

# Pipeline B now waits for Pipeline A to finish before starting
# No competing locks, no manual intervention
```

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **No DynamoDB table created** | S3 backend config has `dynamodb_table` but table doesn't exist → silently no locking | Create the table first in a bootstrap config |
| **force-unlock while apply is running** | Corrupts state — two concurrent writers | Verify the original process is dead before force-unlock |
| **Local state in a team** | Default backend stores state as `terraform.tfstate` locally → no locking, no sharing | Always use remote backend for any shared infrastructure |
| **State file committed to Git** | `.gitignore` missing `*.tfstate` → secrets exposed in Git history | Add to `.gitignore`; use `git filter-repo` to purge if already committed |
| **`terraform state rm` thinking it deletes infra** | Resource removed from state but still running → orphaned resource | Understand: `state rm` ≠ destroy. Use `terraform destroy -target` to actually delete |
| **Serial number conflicts on push** | `state push` fails if state serial is behind → another apply happened | Pull latest state, re-apply changes, push again |

---

## ✅ Best Practices

- **Always use a remote backend** with locking for any shared or production infrastructure — local state is only for personal experiments
- **S3 + DynamoDB** is the de-facto standard for AWS: encrypted bucket + versioning + DynamoDB lock table
- **Enable S3 versioning** on the state bucket — every apply creates a new version, enabling recovery from corruption
- **Encrypt state** with a customer-managed KMS key — state contains plaintext secrets
- **Restrict state access** via S3 bucket policy — only the CI role and designated admins should have access
- **Use `concurrency` groups in CI** to serialize Terraform runs — prevents lock contention across pipelines
- **Never use `-lock=false`** in production — the risk of state corruption far outweighs the inconvenience of waiting
- **`terraform state mv` before renaming resources** — prevents unnecessary destroy/recreate of real infrastructure

---

## 📋 Quick Reference

```bash
# Check if state is locked (try a plan — error reveals lock info)
terraform plan

# Force-unlock a stuck lock (after verifying the original process is dead)
terraform force-unlock <LOCK_ID>

# State inspection
terraform state list                    # All managed resources
terraform state show aws_instance.web  # Details of one resource
terraform state pull > backup.json      # Download state

# State surgery
terraform state mv aws_instance.old aws_instance.new   # Rename
terraform state rm aws_instance.legacy                  # Forget (don't destroy)
terraform import aws_instance.web i-0abc123             # Adopt existing resource

# Detect drift
terraform apply -refresh-only          # Sync state to real world

# DynamoDB: check current lock
aws dynamodb scan --table-name terraform-state-lock

# S3: list state file versions
aws s3api list-object-versions \
  --bucket my-tf-state --prefix prod/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]' --output table
```

---

> **Interview Takeaway:** State locking prevents the most catastrophic Terraform failure mode — concurrent applies corrupting state. Know the S3 + DynamoDB setup cold (it's the most common interview topic). Know when `force-unlock` is safe (original process confirmed dead, state backup taken, second engineer confirms) and when it isn't. And go beyond locking: `state mv` for rename-without-recreate, `state rm` for removing without destroying, and `import` for adopting existing resources — these are the state commands that come up in real production incidents.
