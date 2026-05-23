# Linux Interview Question 1 — EC2 Disk Space Management

> **Topic:** Diagnosing and resolving disk space issues on EC2 instances
> **Level:** Beginner–Intermediate
> **Relevance:** Very common in DevOps/SRE interviews — practically every engineer has hit this in production

---

## ❓ The Question

> **"Your EC2 instance is running out of disk space. How do you troubleshoot and resolve it?"**

This looks like a simple question but interviewers use it to test your **systematic thinking** — do you blindly resize the volume, or do you diagnose first, understand which volume is affected, and then act accordingly?

---

## 🧠 Why This Matters

An EC2 instance typically has at least two EBS volumes:

- **Root EBS Volume** — holds the OS (`/`, `/boot`, `/var`, etc.). If this fills up, the system can **crash or become unresponsive** because Linux itself needs free space to write logs, temp files, PID files, and socket files.
- **Application EBS Volume** — holds app-specific data (logs, JARs, DB data). Filling this up usually breaks the application but the OS stays alive.

The **urgency and response strategy differ completely** depending on which volume is full — that's the first thing to establish.

---

## 🖼️ Reference Diagram (Source: KodeKloud)

![EC2 disk space management — EBS volumes, root and application volumes, troubleshooting steps](https://kodekloud.com/kk-media/image/upload/v1752873382/notes-assets/images/DevOps-Interview-Preparation-Course-Linux-Question-1/ec2-disk-space-management-notes.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **EBS** | Elastic Block Store — persistent block storage volumes attached to EC2 instances |
| **Root Volume** | The primary EBS volume (usually `/dev/xvda` or `/dev/nvme0n1`) containing the OS |
| **Application Volume** | Secondary EBS volume mounted at a custom path (e.g., `/app`, `/data`) |
| **EBS Snapshot** | Point-in-time backup of an EBS volume — stored in S3, used for recovery or resize |
| **`df`** | Disk Free — shows filesystem usage per mount point |
| **`du`** | Disk Usage — shows size of files and directories |
| **`lsblk`** | Lists block devices — shows all EBS volumes and their mount points |
| **`inode`** | File system metadata slot — a disk can have space but still be "full" if inodes run out |
| **`resize2fs` / `xfs_growfs`** | Filesystem resize commands used after expanding an EBS volume |

---

## 🗣️ How to Answer in an Interview

When this comes up, structure your answer around two phases: **diagnose first, then act**.

### Phase 1 — Diagnose

**Step 1: Check overall disk usage across all mount points**

```bash
df -hT
```

```
Filesystem      Type  Size  Used Avail Use% Mounted on
/dev/xvda1      ext4   20G   20G     0 100% /
/dev/xvdb1      ext4  100G   45G   55G  45% /app
tmpfs           tmpfs  2.0G     0  2.0G   0% /dev/shm
```

This immediately tells you **which volume is full** and **what filesystem type** it uses (important for choosing the right resize command later).

**Step 2: List all attached block devices**

```bash
lsblk
```

```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda        202:0    0   20G  0 disk
└─xvda1     202:1    0   20G  0 part /
xvdb        202:16   0  100G  0 disk
└─xvdb1     202:17   0  100G  0 part /app
```

This confirms which EBS volumes exist and where they're mounted.

**Step 3: Find what's consuming space**

```bash
# Top 20 largest files/directories under root
du -ahx / | sort -rh | head -20

# Focus on known problem areas
du -sh /var/log/*          # Log files
du -sh /tmp/*              # Temp files
du -sh /home/*             # User directories
du -sh /app/*              # Application data

# Check for large files that might be open but already deleted
# (They won't show up in du but still hold space)
lsof +L1 | grep deleted
```

The `lsof +L1` command is a **key interview differentiator** — files deleted while a process has them open still consume disk space until the process releases the file handle. This is a very common production gotcha.

**Step 4: Check inode usage (not just space)**

```bash
df -ih
```

```
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/xvda1      1.3M  1.3M     0  100% /
```

A disk can show **50% space available but 100% inode usage** — creating any new file will fail even though bytes are free. This happens on filesystems hosting many small files (e.g., a mail spool, npm cache, or container layer cache).

---

### Phase 2 — Respond Based on Which Volume is Affected

```
Root Volume full?          → Emergency. Free space immediately.
Application Volume full?   → Snapshot first, then resize.
```

---

## ⚙️ Root Volume Full — Resolution

The OS depends on the root volume. At 100%, critical operations fail: SSH logins may fail, systemd can't write PID files, cron jobs fail silently. **Act fast.**

```bash
# 1. Clear system logs (safe to truncate, not delete)
sudo truncate -s 0 /var/log/syslog
sudo journalctl --vacuum-size=200M    # Keep only 200MB of journal logs
sudo journalctl --vacuum-time=3d      # Or keep only last 3 days

# 2. Clean package manager cache
sudo apt-get clean        # Debian/Ubuntu
sudo yum clean all        # RHEL/Amazon Linux

# 3. Remove old kernel versions (common culprit on long-running instances)
sudo apt-get autoremove --purge       # Ubuntu
sudo package-cleanup --oldkernels --count=1  # Amazon Linux / RHEL

# 4. Find and remove core dumps
find / -name "core" -type f -size +100M 2>/dev/null

# 5. Clear Docker junk (if Docker is running)
docker system prune -a --volumes      # Removes stopped containers, unused images

# 6. Kill processes holding deleted files
# From lsof +L1 output — note the PID and restart the service
sudo systemctl restart <service-name>
```

After freeing space, **check the root cause** — if `/var/log` filled up, set up log rotation. If it's Docker, implement image lifecycle policies.

---

## ⚙️ Application Volume Full — Resolution

You have more breathing room here. The OS is alive. **Take a snapshot before touching anything.**

```bash
# Step 1: Create EBS snapshot via AWS CLI (before any changes)
aws ec2 create-snapshot \
  --volume-id vol-0abc123def456789 \
  --description "Pre-resize snapshot $(date +%Y-%m-%d)"

# Step 2: Identify what's consuming space
du -sh /app/* | sort -rh | head -20

# Common culprits:
# - Old deployment JARs/WARs
ls -lth /app/releases/ | head -20
# - Application logs
du -sh /app/logs/*
# - Temporary upload directories
du -sh /app/tmp/*

# Step 3: Clean up safely
# Remove all but the last 3 deployment artifacts
ls -t /app/releases/ | tail -n +4 | xargs -I{} rm -rf /app/releases/{}

# Compress old logs instead of deleting them
gzip /app/logs/*.log.1 /app/logs/*.log.2
```

If cleanup isn't enough, **resize the EBS volume:**

```bash
# Step 1: Resize the volume in AWS (no downtime required for modern EBS)
aws ec2 modify-volume \
  --volume-id vol-0abc123def456789 \
  --size 200      # Increase from 100GB to 200GB

# Step 2: Wait for the modification to complete
aws ec2 describe-volumes-modifications \
  --volume-id vol-0abc123def456789

# Step 3: Extend the partition (if needed)
sudo growpart /dev/xvdb 1

# Step 4: Resize the filesystem
# For ext4:
sudo resize2fs /dev/xvdb1

# For XFS (used by Amazon Linux 2):
sudo xfs_growfs /app

# Verify
df -hT /app
```

The resize is **online** — no reboot required. The filesystem sees the new space immediately after `resize2fs` / `xfs_growfs`.

---

## 🔐 Security Perspective

- **Never skip the snapshot**: EBS snapshots are cheap and incremental. A resize gone wrong (wrong partition, wrong filesystem) can be catastrophic without a snapshot.
- **Audit what you're deleting**: Before `rm -rf`, always `ls` the target first. Running `du` output into `xargs rm` without checking is a common cause of accidental data loss.
- **Log rotation as a security control**: Excessive log accumulation can be a sign of an attack (e.g., a brute-force flood filling `/var/log/auth.log`). Before truncating, review what the logs contain.
- **IAM permissions for EBS operations**: The EC2 instance role or the user running `aws ec2 modify-volume` needs `ec2:ModifyVolume` and `ec2:DescribeVolumesModifications` — not full `ec2:*`.

---

## 🌍 Real-World Scenario

**Situation:** A production Java application on EC2 stopped accepting requests at 2 AM. Monitoring showed no CPU or memory spikes. Alert: disk usage 100%.

**Root Cause:** The application was deployed 6 months ago with log4j configured to write DEBUG-level logs. Nobody noticed because the volume was 100GB. A traffic spike generated 60GB of debug logs overnight.

**Resolution (live, no downtime):**
1. `df -hT` → `/app` at 100%
2. `du -sh /app/logs/*` → `app-debug.log` = 58GB
3. `truncate -s 0 /app/logs/app-debug.log` → freed 58GB immediately (app had the file open, so delete would have left it in place)
4. Application recovered automatically (Java retried the failed log write)
5. Next day: changed log4j config to INFO level, added logrotate config, set CloudWatch disk utilization alarm at 80%

**Lesson:** `truncate -s 0` (not `rm`) is safer when a process has the file open. And monitoring should have caught 80% usage long before 100%.

---

## ⚠️ Common Gotchas

| Gotcha | What Happens | Fix |
|--------|-------------|-----|
| **Deleting open files** | `rm app.log` while app is writing — space not freed until process restarts | Use `truncate -s 0 app.log` instead |
| **Inode exhaustion** | `df -h` shows space available but writes fail | Check `df -ih`; find many-small-file culprit (Docker, npm) |
| **Forgetting `growpart`** | `resize2fs` fails after EBS resize | On partitioned volumes, run `growpart` before `resize2fs` |
| **XFS vs ext4 resize** | `resize2fs` on an XFS filesystem errors out | Check `df -T` for filesystem type; use `xfs_growfs` for XFS |
| **Wrong volume ID in snapshot** | Snapshot taken on wrong volume | Cross-check `lsblk` mount points with `aws ec2 describe-volumes` |
| **Tmpfs counts as full** | `/run` or `/dev/shm` at 100% causes false alarm | `df -hT` — recognize `tmpfs` type; it lives in RAM, not EBS |

---

## ✅ Best Practices

- Set CloudWatch alarms at **80% disk usage** — never let it hit 100% in production
- Implement **logrotate** for all application logs with `daily`, `rotate 7`, `compress` options
- Use **EBS volume tagging** so you always know which volume belongs to which service
- Automate **EBS snapshot lifecycle** with AWS Data Lifecycle Manager (DLM) — daily snapshots retained for 7 days
- For applications that generate large files (video processing, ML training), use **S3 or EFS** instead of EBS to avoid fixed-size volume constraints
- In Docker environments, run `docker system prune` on a cron schedule (e.g., weekly) to prevent image/layer accumulation

---

## 🔄 Quick Comparison: Root vs Application Volume Actions

| Aspect | Root Volume (`/`) | Application Volume (`/app`) |
|--------|------------------|----------------------------|
| **Priority** | Critical — act immediately | Important — slightly more time |
| **Snapshot first?** | If possible, but may not be | Always |
| **Safe cleanup targets** | `/var/log`, `/tmp`, package cache, old kernels | App logs, old JARs/WARs, temp upload dirs |
| **Resize possible?** | Yes — online, no reboot | Yes — online, no reboot |
| **Filesystem resize cmd** | `resize2fs` (ext4) / `xfs_growfs` (XFS) | Same |
| **OS impact if full** | SSH failures, systemd errors, possible crash | Application breaks, OS survives |

---

## 📋 Quick Reference Commands

```bash
# Diagnose
df -hT                          # Disk usage by mount point (with filesystem type)
df -ih                          # Inode usage
lsblk                           # All block devices
du -ahx / | sort -rh | head -20 # Largest items under root
lsof +L1 | grep deleted         # Files deleted but still held open

# Free space (root volume emergency)
sudo journalctl --vacuum-size=200M
sudo apt-get clean / yum clean all
docker system prune -a

# Resize EBS (application volume)
aws ec2 create-snapshot --volume-id vol-xxx --description "pre-resize"
aws ec2 modify-volume --volume-id vol-xxx --size 200
sudo growpart /dev/xvdb 1       # Extend partition
sudo resize2fs /dev/xvdb1       # ext4
sudo xfs_growfs /app            # XFS

# Verify
df -hT /app
```

---

> **Interview Takeaway:** Don't jump to "resize the volume." Show the interviewer you diagnose first (`df`, `du`, `lsof`), distinguish root vs application volume urgency, clean up before resizing when possible, and always snapshot before making changes. That systematic approach is what separates a senior engineer from someone who just throws more storage at the problem.
