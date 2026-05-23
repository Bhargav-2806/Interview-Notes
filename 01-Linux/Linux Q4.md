# Linux Interview Question 4 — Automated Log Transfer to S3 with Bash & Cron

> **Topic:** Design a Linux script that transfers log files to an S3 bucket at scheduled intervals
> **Level:** Beginner–Intermediate
> **Relevance:** Tests Linux scripting + AWS CLI + automation thinking — a very practical DevOps combination

---

## ❓ The Question

> **"How would you design a solution to automatically transfer log files from a Linux server to an S3 bucket at regular intervals?"**

This is a design + implementation question. The interviewer wants to see that you can combine three things: **Bash scripting**, **AWS authentication best practices**, and **Linux task scheduling (cron)**. A complete answer covers all three, not just the `aws s3 cp` command.

---

## 🖼️ Reference Diagram (Source: KodeKloud)

![High-level design — IAM roles, access keys, Bash script, AWS CLI, cron job, S3 bucket](https://kodekloud.com/kk-media/image/upload/v1752873388/notes-assets/images/DevOps-Interview-Preparation-Course-Linux-Question-4/aws-s3-iam-design-notes.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **IAM Role** | AWS identity attached to an EC2 instance — grants permissions without storing keys |
| **IAM Access Keys** | Access Key ID + Secret Access Key pair used for programmatic AWS access |
| **Instance Metadata** | EC2 service at `169.254.169.254` that provides temporary credentials from the attached IAM role |
| **Shebang (`#!/bin/bash`)** | First line of a shell script — tells the OS which interpreter to use |
| **`aws s3 cp`** | AWS CLI command to copy a single file to/from S3 |
| **`aws s3 sync`** | AWS CLI command to sync a directory — only copies new/changed files |
| **Cron** | Linux time-based job scheduler — runs commands at defined intervals |
| **Crontab** | The file that stores cron job definitions |
| **Log Rotation** | Process of archiving and compressing old logs so they don't fill disk |

---

## 🔐 Step 1 — Authentication: IAM Role vs Access Keys

Before the script even runs, the server needs permission to write to S3. There are two ways to do this, and **the interviewer will care which one you recommend**.

### Option A: IAM Role (Recommended — Always Use This on EC2)

Attach an IAM role to the EC2 instance with an S3 write policy. The AWS CLI automatically picks up temporary credentials from the EC2 instance metadata — no keys stored anywhere.

```json
// IAM Policy: allow writing to a specific S3 bucket (least privilege)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-log-bucket",
        "arn:aws:s3:::my-log-bucket/*"
      ]
    }
  ]
}
```

Attach this policy to a role → attach the role to the EC2 instance. No keys, no rotation, no risk of credential leaks.

### Option B: IAM Access Keys (Use Only When IAM Roles Aren't Possible)

For non-EC2 servers (on-premise, other clouds), configure access keys via AWS CLI:

```bash
aws configure
# AWS Access Key ID: AKIA...
# AWS Secret Access Key: ****
# Default region: us-east-1
# Default output format: json

# This stores credentials at ~/.aws/credentials
# OR use environment variables (better for scripts):
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-1"
```

**Always prefer IAM roles on EC2.** Access keys stored in files can be accidentally committed to Git, exposed via `cat`, or leaked in process listings. The IAM role approach has none of these risks.

---

## ⚙️ Step 2 — The Bash Script

### Basic Version

```bash
#!/bin/bash
# log-to-s3.sh — Transfer application logs to S3
# Usage: ./log-to-s3.sh

LOG_SOURCE="/var/log/application/application-01"
S3_DEST="s3://my-log-bucket/logs/$(hostname)/"

aws s3 cp "$LOG_SOURCE" "$S3_DEST"
```

This works but is fragile — no error handling, no logging, no timestamp. In an interview, mention you'd go further.

### Production-Ready Version

```bash
#!/bin/bash
# ============================================================
# log-to-s3.sh
# Transfers application log files to S3 with error handling,
# logging, and notification on failure.
# ============================================================

set -euo pipefail  # Exit on error, undefined vars, pipe failures

# ---- Configuration ----
LOG_SOURCE_DIR="/var/log/application"
LOG_FILE_PATTERN="application-*.log"
S3_BUCKET="my-log-bucket"
S3_PREFIX="logs/$(hostname)/$(date +%Y/%m/%d)"
SCRIPT_LOG="/var/log/log-transfer.log"
TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")

# ---- Logging helper ----
log() {
  echo "[$TIMESTAMP] $1" | tee -a "$SCRIPT_LOG"
}

# ---- Pre-flight checks ----
if ! command -v aws &>/dev/null; then
  log "ERROR: AWS CLI not found. Install with: sudo yum install aws-cli"
  exit 1
fi

if [ ! -d "$LOG_SOURCE_DIR" ]; then
  log "ERROR: Log directory $LOG_SOURCE_DIR does not exist"
  exit 1
fi

# ---- Verify AWS credentials work ----
if ! aws sts get-caller-identity &>/dev/null; then
  log "ERROR: AWS credentials not configured or IAM role not attached"
  exit 1
fi

# ---- Transfer logs ----
log "INFO: Starting log transfer to s3://$S3_BUCKET/$S3_PREFIX/"

# Use sync to only transfer new/changed files (idempotent)
if aws s3 sync "$LOG_SOURCE_DIR" "s3://$S3_BUCKET/$S3_PREFIX/" \
    --exclude "*" \
    --include "$LOG_FILE_PATTERN" \
    --storage-class STANDARD_IA \
    --sse AES256 \
    2>> "$SCRIPT_LOG"; then
  log "INFO: Log transfer completed successfully"
else
  log "ERROR: Log transfer failed — check AWS CLI output above"
  # Optional: send alert (SNS, Slack, PagerDuty)
  aws sns publish \
    --topic-arn "arn:aws:sns:us-east-1:123456789:ops-alerts" \
    --message "Log transfer failed on $(hostname) at $TIMESTAMP" \
    --subject "Log Transfer Failure" 2>/dev/null || true
  exit 1
fi

# ---- Optional: Clean up local logs older than 7 days ----
# Only after confirming successful transfer
find "$LOG_SOURCE_DIR" -name "$LOG_FILE_PATTERN" -mtime +7 -delete
log "INFO: Cleaned up log files older than 7 days"
```

### Key Design Decisions Worth Mentioning in an Interview

**`set -euo pipefail`** — Without this, a script continues running even after a failed command. `-e` exits on error, `-u` errors on undefined variables, `-o pipefail` catches errors in pipes like `cmd1 | cmd2`.

**`aws s3 sync` over `aws s3 cp`** — `sync` is idempotent: it only transfers files that don't already exist in S3 or have changed. If the cron job runs again due to a retry, it won't re-upload gigabytes of already-transferred logs.

**`--storage-class STANDARD_IA`** — Log files are written once and rarely read. `STANDARD_IA` (Infrequent Access) costs ~40% less than Standard for storage. For logs older than 30 days, `GLACIER` is even cheaper.

**`--sse AES256`** — Enables server-side encryption on S3. Most compliance frameworks (SOC 2, PCI) require encryption at rest.

**Date-based S3 prefix** `logs/$(hostname)/2025/01/15/` — Organizes logs by server and date, making them searchable with Athena or easy to find manually.

---

## ⏰ Step 3 — Scheduling with Cron

### Crontab Syntax

```
┌──── minute (0-59)
│ ┌──── hour (0-23)
│ │ ┌──── day of month (1-31)
│ │ │ ┌──── month (1-12)
│ │ │ │ ┌──── day of week (0-6, 0=Sunday)
│ │ │ │ │
* * * * *  command
```

### Common Schedules

```bash
# Edit crontab for the current user
crontab -e

# Run every hour
0 * * * * /opt/scripts/log-to-s3.sh

# Run every 15 minutes
*/15 * * * * /opt/scripts/log-to-s3.sh

# Run daily at 2 AM (good for daily log batches)
0 2 * * * /opt/scripts/log-to-s3.sh

# Run at midnight on the 1st of every month
0 0 1 * * /opt/scripts/log-to-s3.sh

# With output logging (so you can see if cron ran)
0 * * * * /opt/scripts/log-to-s3.sh >> /var/log/log-transfer-cron.log 2>&1
```

### Make the Script Executable First

```bash
chmod +x /opt/scripts/log-to-s3.sh

# Verify cron is running
systemctl status crond    # RHEL/Amazon Linux
systemctl status cron     # Debian/Ubuntu

# Check cron logs to confirm execution
sudo grep CRON /var/log/syslog | tail -20     # Ubuntu/Debian
sudo grep crond /var/log/messages | tail -20  # RHEL/Amazon Linux
```

### Cron PATH Issue (Classic Gotcha)

Cron runs with a minimal `PATH` — it won't find `aws` if it's installed in `/usr/local/bin`. Always use the full path in cron jobs:

```bash
# Find where aws is installed
which aws
# /usr/local/bin/aws

# Use full path in crontab
0 * * * * /usr/local/bin/aws s3 sync /var/log/app s3://my-bucket/logs/

# OR set PATH at the top of the crontab file
PATH=/usr/local/bin:/usr/bin:/bin
0 * * * * /opt/scripts/log-to-s3.sh
```

---

## 🌍 Real-World Scenario

**Situation:** A microservices app on EC2 generates 2GB of logs per day per instance (10 instances = 20GB/day). The team needs logs centralized in S3 for analysis with Athena. Storing 30 days of logs locally would require 600GB of EBS per instance.

**Solution implemented:**
```bash
# Cron: every hour, sync last hour's logs to S3 and clean up local files > 24h
0 * * * * /opt/scripts/log-to-s3.sh

# S3 lifecycle policy: move to STANDARD_IA after 30 days, GLACIER after 90 days
# (set in Terraform, not manually)
```

```hcl
# Terraform: S3 lifecycle for log bucket
resource "aws_s3_bucket_lifecycle_configuration" "log_lifecycle" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "log-tiering"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365    # Delete after 1 year
    }
  }
}
```

**Result:** EBS volumes dropped from needing 600GB to 50GB per instance (24 hours of logs only). S3 costs were 60% lower than EBS for the same data volume. Logs queryable via Athena with no infrastructure to manage.

---

## 🔄 `aws s3 cp` vs `aws s3 sync`

| Command | Behaviour | Best For |
|---------|-----------|---------|
| `aws s3 cp file.log s3://bucket/` | Copies one specific file | Known single file, one-time transfer |
| `aws s3 cp /dir/ s3://bucket/ --recursive` | Copies entire directory | Full backup of a directory |
| `aws s3 sync /dir/ s3://bucket/` | Only copies new or changed files | Recurring scheduled transfers — idempotent, efficient |
| `aws s3 sync --delete` | Sync + removes files in S3 not in local | Mirror mode — use carefully |

For a cron-based log transfer, **`aws s3 sync` is always the right choice** — it skips already-uploaded files, making reruns safe and fast.

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **Cron PATH doesn't include `aws`** | Script works manually but fails in cron | Use full path `/usr/local/bin/aws` or set PATH in crontab |
| **`aws s3 cp` on active log file** | Copies a partial log file mid-write | Use `sync` with completed log files; configure log rotation to roll files first |
| **No IAM permissions** | Silent failure or `AccessDenied` error | Test manually as the same user cron runs as; verify IAM policy |
| **Script not executable** | Cron silently fails — `Permission denied` | `chmod +x /opt/scripts/log-to-s3.sh` |
| **S3 bucket in wrong region** | Slow transfers; potential cross-region costs | Use `--region` flag or ensure bucket is in the same region as EC2 |
| **No error notification** | Script fails silently for weeks | Add SNS/Slack alert on failure; check cron logs regularly |
| **Uploading logs with secrets** | App logs sometimes contain API keys, tokens | Scrub sensitive patterns before upload; use S3 bucket policy to restrict access |

---

## ✅ Best Practices

- **Always use IAM roles on EC2** — never hardcode or store access keys in scripts or files
- Use **`aws s3 sync`** for recurring transfers — it's idempotent and skips unchanged files
- **Organize S3 paths** with hostname + date prefix: `logs/web-server-01/2025/01/15/` — makes retrieval and Athena querying efficient
- Enable **server-side encryption** (`--sse AES256`) on all log uploads — required by most compliance frameworks
- Set an **S3 lifecycle policy** to automatically move old logs to STANDARD_IA → GLACIER → expire
- Always log the **script's own output** to a file — if the transfer fails silently, you need evidence
- Test the script manually as the **same user** cron will use (usually `root` or `ec2-user`) before scheduling it

---

## 📋 Quick Reference

```bash
# Make script and schedule it
chmod +x /opt/scripts/log-to-s3.sh
crontab -e
# Add: 0 * * * * /opt/scripts/log-to-s3.sh >> /var/log/log-transfer.log 2>&1

# Test AWS access
aws sts get-caller-identity
aws s3 ls s3://my-log-bucket/

# Manual sync test
aws s3 sync /var/log/application/ s3://my-log-bucket/logs/test/ \
  --exclude "*" --include "*.log" --dryrun

# Verify cron ran
grep CRON /var/log/syslog | grep log-to-s3 | tail -10

# Check script log
tail -50 /var/log/log-transfer.log
```

---

> **Interview Takeaway:** Don't just say "use `aws s3 cp` and a cron job." Show you've thought about all three layers: **authentication** (IAM role, not access keys on EC2), **the script itself** (`set -euo pipefail`, error handling, `sync` not `cp`, encryption), and **scheduling** (cron syntax, PATH gotcha, output logging). Mentioning S3 lifecycle policies for cost optimization is the detail that elevates a good answer to a great one.
