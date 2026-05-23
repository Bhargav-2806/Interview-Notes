# Linux Interview Question 3 — EC2 Instances Terminating in an Auto Scaling Group

> **Topic:** Debugging EC2 instances that keep getting terminated by an ASG due to unhealthy status
> **Level:** Intermediate
> **Relevance:** A classic DevOps production scenario — shows whether you can think through an infinite termination loop systematically

---

## ❓ The Question

> **"Multiple EC2 instances in your Auto Scaling Group are being repeatedly terminated, causing application downtime. EC2 pricing, quotas, and ASG configuration all look correct. How do you debug this?"**

This is a **scenario-based troubleshooting question**. The interviewer already told you the ASG config is fine — so they're testing whether you can pivot to the next logical layer: the health of the instances themselves.

---

## 🧠 Understanding the Problem First

Before diving into commands, you need to understand **why the ASG is terminating instances**. An ASG terminates an instance for one of two reasons:

1. **Scale-in** (planned): Demand dropped, ASG is reducing capacity → not the issue here
2. **Unhealthy replacement** (unplanned): The ASG health check marked the instance as unhealthy → this is your culprit

The ASG checks instance health in two ways:
- **EC2 status check** (default): Did the hypervisor/networking fail? (hardware-level)
- **ELB health check** (optional but common): Is the app responding on the health check endpoint?

If the app is crashing or becoming unresponsive shortly after launch, the ELB health check will fail → ASG marks it unhealthy → terminates it → launches a replacement → same thing happens again.

**This is the termination loop:**

```
ASG launches new EC2
       ↓
Instance starts, app crashes (resource exhaustion / app bug)
       ↓
ELB health check fails → ASG marks instance unhealthy
       ↓
ASG terminates instance → launches replacement
       ↓
          (repeat indefinitely)
```

---

## 🖼️ Reference Diagrams (Source: KodeKloud)

![EC2 instances in ASG being terminated — debugging the issue](https://kodekloud.com/kk-media/image/upload/v1752873385/notes-assets/images/DevOps-Interview-Preparation-Course-Linux-Question-3/ec2-instances-asg-debugging-issue.jpg)

![EC2 instance issues — full disk, high CPU, no memory](https://kodekloud.com/kk-media/image/upload/v1752873386/notes-assets/images/DevOps-Interview-Preparation-Course-Linux-Question-3/ec2-instance-issues-handwritten-note.jpg)

![ASG → EC2 → Application → Unhealthy → Terminate loop flowchart](https://kodekloud.com/kk-media/image/upload/v1752873387/notes-assets/images/DevOps-Interview-Preparation-Course-Linux-Question-3/asg-ec2-application-flowchart.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **ASG** | Auto Scaling Group — manages a fleet of EC2 instances, replacing unhealthy ones automatically |
| **Health Check** | ASG periodically checks if each instance is healthy (EC2 status check or ELB check) |
| **Unhealthy threshold** | Number of consecutive failed checks before ASG marks instance as unhealthy |
| **Grace period** | Time the ASG waits after launch before starting health checks (gives the app time to start) |
| **Termination policy** | Rules for which instance ASG picks to terminate first (oldest, newest, closest to billing hour) |
| **OOM Killer** | Linux kernel mechanism that kills processes when RAM is exhausted |
| **Swap** | Disk-backed memory overflow — when RAM is full, kernel uses swap (much slower) |
| **CloudWatch** | AWS monitoring service — where you pull EC2 CPU/memory/disk metrics |

---

## 🗣️ How to Answer in an Interview

The key here is to **show a structured debugging process**. Don't just list tools — walk through the logical chain of investigation.

### Step 1 — Buy Yourself Time to Investigate

Before you can debug, you need a live instance to look at. If the ASG is terminating instances within minutes, you may need to **temporarily suspend the health check replacement** so an instance stays up long enough to inspect.

```bash
# Suspend the ReplaceUnhealthy process temporarily
aws autoscaling suspend-processes \
  --auto-scaling-group-name my-asg \
  --scaling-processes ReplaceUnhealthy

# ⚠️ Remember to re-enable after debugging
aws autoscaling resume-processes \
  --auto-scaling-group-name my-asg \
  --scaling-processes ReplaceUnhealthy
```

You can also increase the **health check grace period** to give yourself more time:

```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --health-check-grace-period 600    # 10 minutes instead of default
```

### Step 2 — Check ASG Activity History

Before SSHing into anything, check what ASG is actually saying about why instances are being terminated:

```bash
# Check ASG activity log
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-asg \
  --max-items 20

# You'll see entries like:
# "Terminating EC2 instance: i-0abc123. Reason: Instance failed ELB health checks for 3 consecutive attempts."
# OR
# "Terminating EC2 instance: i-0abc123. Reason: Instance status check failed."
```

This tells you **whether it's an EC2-level failure or an app-level failure** — which focuses your next steps.

### Step 3 — SSH In and Check the Three Usual Suspects

Log into a problematic instance (while it's still alive) and check the three most common causes:

---

#### Suspect 1: CPU Exhaustion

```bash
# Real-time process view — sort by CPU
top

# Non-interactive: snapshot of top CPU consumers
ps aux --sort=-%cpu | head -20

# If you suspect a Java app:
# High CPU in a Java process often means GC thrashing (heap too small)
# or an infinite loop / deadlock
ps aux | grep java
jstack <PID> | grep -A 10 "runnable"    # thread dump to spot stuck threads

# Historical CPU: was it always high or did it spike?
# Check CloudWatch CPU metric for the instance
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average
```

**What to look for:** A single process at 99% CPU means an app-level bug (infinite loop, GC thrash, hot thread). Widespread high CPU across many processes suggests the instance type is undersized.

---

#### Suspect 2: Disk Space Full

```bash
# Check all mount points
df -hT

# Find the space hogs
du -ahx / | sort -rh | head -20

# Common culprits in ASG instances:
du -sh /var/log/*          # Application or system logs
du -sh /tmp/*              # Temp files from deployments
du -sh /opt/app/logs/*     # App-specific log dirs

# Files deleted but still held open (won't free space until process restarts)
lsof +L1 | grep deleted

# Inode exhaustion (disk has space but can't create new files)
df -ih
```

In ASG environments specifically, a common disk issue is **log files accumulating between deployments**. The instance starts fresh but the app immediately starts writing logs. If log rotation isn't configured, a busy instance fills up fast.

---

#### Suspect 3: Memory Exhaustion

```bash
# Current memory and swap state
free -m

# Example output showing a problem:
#               total   used   free  shared  buff/cache  available
# Mem:           7983   7940      10      45          33         10
# Swap:             0      0       0
# ↑ Available memory: 10MB, no swap → OOM killer about to fire

# Who is consuming memory?
ps aux --sort=-%mem | head -20

# Check if OOM killer has already fired (killed processes)
dmesg | grep -i "oom\|killed process\|out of memory"
# or
sudo journalctl -k | grep -i oom

# Check swap usage in detail
swapon --show
cat /proc/swaps
```

**What `free -m` output means:**

```
             total    used    free   available
Mem:          8000    7900     100        50   ← 50MB available = critical
Swap:         2000    2000       0         0   ← swap also exhausted = OOM imminent
```

When both RAM and swap hit zero, the Linux **OOM killer** starts terminating processes. It often kills the main application process — which causes the health check to fail — which triggers ASG termination.

---

### Step 4 — Check Application Logs

System resources may look fine but the app itself could be crashing:

```bash
# Check app logs
sudo tail -200 /var/log/app/application.log
sudo journalctl -u myapp.service -n 200 --no-pager

# Look for crash patterns
grep -i "error\|exception\|fatal\|crash\|killed\|oom" /var/log/app/*.log | tail -50

# Check if the app process is even running
systemctl status myapp
ps aux | grep myapp

# Check what systemd says about restart attempts
systemctl show myapp | grep -E "NRestarts|ActiveState|SubState"
```

### Step 5 — Check the Health Check Endpoint Directly

Simulate what the ELB is doing when it marks the instance unhealthy:

```bash
# From inside the instance, curl the health check endpoint
curl -v http://localhost:8080/health
curl -v http://localhost:8080/actuator/health  # Spring Boot

# From outside (on your laptop), check if it's reachable at all
curl -v http://<INSTANCE_PRIVATE_IP>:8080/health

# Check if the port is even open
ss -tlnp | grep 8080
netstat -tlnp | grep 8080
```

If the health endpoint returns 500, check app logs. If it times out, the app may be starting slowly — increase the ASG grace period.

---

## 🔄 Analysis → Action Table

| Root Cause | Signal | Immediate Action | Long-term Fix |
|-----------|--------|-----------------|---------------|
| **High CPU** | `top` shows single process at 99% | Coordinate with dev team to fix thread/GC issue | Right-size instance; tune JVM heap; add CPU alarm |
| **Full disk** | `df -h` shows 100% on `/` or `/var` | `journalctl --vacuum-size=200M`; clear logs | Configure logrotate; add disk alarm at 80% |
| **Memory exhaustion** | `free -m` shows near-zero available | Restart app process; check for memory leak | Upgrade instance type; fix memory leak; add swap |
| **App crash loop** | systemd shows repeated restarts | Check app logs for exception cause | Fix underlying bug; add proper health check endpoint |
| **Slow startup** | Health check fails before app is ready | Increase ASG grace period | Optimize app startup; add readiness probe concept |

---

## 🔐 Security & Operational Perspective

- **Suspend `ReplaceUnhealthy` with caution**: Disabling auto-replacement means a failed instance stays failed — keep the window short and re-enable immediately after debugging
- **Enable Instance Protection** on a specific instance if you need to investigate without the ASG killing it
- **CloudWatch agent for memory/disk metrics**: By default, AWS CloudWatch doesn't expose memory or disk metrics — you need to install the CloudWatch agent on your AMI to get them. Without it, you're flying blind on two of the three main culprits
- **ASG Instance Refresh** vs manual termination: Don't manually terminate instances to "fix" the loop — fix the root cause, then let the ASG naturally replace with healthy instances

```bash
# Protect a specific instance from ASG termination (for investigation)
aws autoscaling set-instance-protection \
  --instance-ids i-0abc123 \
  --auto-scaling-group-name my-asg \
  --protected-from-scale-in

# Remove protection when done
aws autoscaling set-instance-protection \
  --instance-ids i-0abc123 \
  --auto-scaling-group-name my-asg \
  --no-protected-from-scale-in
```

---

## 🌍 Real-World Scenario

**Situation:** A Node.js e-commerce app running in an ASG of 4 instances. During a flash sale, the ops team noticed instances were terminating every 8 minutes. Traffic was high but within the expected load.

**Investigation:**
1. Suspended `ReplaceUnhealthy` on the ASG, protected one instance
2. `free -m` → Available memory: 12MB, Swap: 0 (exhausted)
3. `ps aux --sort=-%mem` → Node.js process consuming 7.8GB on an 8GB instance
4. `dmesg | grep oom` → OOM killer had been firing, killing the Node process
5. Node process death → health check endpoint down → ELB check fails → ASG terminates

**Root cause:** A code change 3 days earlier introduced a memory leak — an event listener was being registered inside a request handler, causing it to multiply on every incoming request. Under normal traffic it was slow; under flash-sale traffic it snowballed in minutes.

**Fix:**
- Immediate: Rolled back the 3-day-old deployment
- Short term: Changed instance type from `t3.large` (8GB) to `r5.large` (16GB) as breathing room
- Long term: Added memory profiling (`--inspect` + clinic.js), CloudWatch memory alarm at 75%, and added the fix in the next deploy

**Lesson:** The ASG was doing exactly what it was supposed to do — terminating unhealthy instances. The problem wasn't the ASG; it was a memory leak at the application layer. Always look at what the ASG is reacting to, not the ASG itself.

---

## ⚠️ Common Gotchas

| Gotcha | What Happens | Fix |
|--------|-------------|-----|
| **No CloudWatch memory/disk metrics** | You can't see memory or disk in CloudWatch dashboards | Install CloudWatch agent in your base AMI |
| **Grace period too short** | App takes 90 seconds to start; grace period is 60 seconds → always unhealthy | Set grace period > app startup time + buffer |
| **OOM kill not obvious** | Instance looks fine in CloudWatch CPU — OOM happens fast | Check `dmesg | grep oom` and `journalctl -k` |
| **Deleting log files on running app** | Space not freed because process still holds file open | Use `truncate -s 0` not `rm`; or restart the process |
| **Forgetting to resume ASG processes** | Suspended `ReplaceUnhealthy` stays suspended in production | Set a calendar reminder or use a script that auto-resumes |
| **Health check endpoint not implemented** | Returns 200 always even when DB is down | Health check should verify actual dependencies (DB, cache) |

---

## ✅ Best Practices

- **Install CloudWatch agent in your base AMI** — memory and disk are the top two ASG termination causes and neither shows up in default CloudWatch metrics
- Set CloudWatch alarms: CPU > 80%, memory > 80%, disk > 80% — alert before the ASG notices
- Build a meaningful **health check endpoint** that verifies the app and its dependencies (DB connection, cache availability), not just `return 200`
- Set the **ASG grace period** to at least 2× your app's typical startup time
- Configure **logrotate** in your AMI — log accumulation is a predictable cause of disk issues on long-running instances
- For Java apps, always set explicit **JVM heap limits** (`-Xmx`) to prevent unbounded memory growth
- Use **ASG lifecycle hooks** for draining connections gracefully before termination — reduces downtime during the replacement cycle

---

## 📋 Quick Reference — Debug Checklist

```bash
# 1. Check ASG activity log
aws autoscaling describe-scaling-activities --auto-scaling-group-name my-asg

# 2. Protect an instance to keep it alive
aws autoscaling set-instance-protection \
  --instance-ids i-0abc123 --auto-scaling-group-name my-asg --protected-from-scale-in

# 3. CPU check
top
ps aux --sort=-%cpu | head -10

# 4. Disk check
df -hT
du -ahx / | sort -rh | head -20
lsof +L1 | grep deleted    # Deleted-but-open files

# 5. Memory check
free -m
dmesg | grep -i oom
ps aux --sort=-%mem | head -10

# 6. App check
systemctl status myapp
sudo journalctl -u myapp -n 100
curl -v http://localhost:8080/health

# 7. Resume ASG when done
aws autoscaling resume-processes \
  --auto-scaling-group-name my-asg --scaling-processes ReplaceUnhealthy
```

---

> **Interview Takeaway:** When an ASG keeps terminating instances, the ASG is almost never the problem — it's doing its job correctly. The question is: *what is it reacting to?* Walk the interviewer through buying investigation time (suspend ReplaceUnhealthy, protect an instance), then check the three resource suspects in order: CPU (`top`), disk (`df -hT`), memory (`free -m`). Then check app logs and the health check endpoint. That systematic chain — rather than jumping straight to "resize the instance" — is what demonstrates senior-level thinking.
