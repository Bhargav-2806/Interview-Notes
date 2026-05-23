# Linux — Medium & Hard Interview Questions

> **Target:** Mid-level DevOps / SRE / Platform Engineers (3–5 years experience)
> **Difficulty:** Medium ⚡ and Hard 🔥
> **Coverage:** Pure Linux + Linux as it relates to Kubernetes and containers
> **Last Updated:** 2025

---

## Table of Contents

| # | Question | Level | Category |
|---|----------|-------|----------|
| Q1 | Linux Process States, Zombies & Orphans | ⚡ Medium | Process Management |
| Q2 | File Permissions — SUID, SGID, Sticky Bit | ⚡ Medium | Security |
| Q3 | Systemd Deep Dive — Units, Targets, Journald | ⚡ Medium | Init System |
| Q4 | Linux Performance Troubleshooting — USE Method | 🔥 Hard | Performance |
| Q5 | Linux Memory — Virtual Memory, OOM Killer, Swap | 🔥 Hard | Memory |
| Q6 | Advanced Networking — iptables, ss, tcpdump | 🔥 Hard | Networking |
| Q7 | Linux Boot Process — BIOS to Userspace | ⚡ Medium | System |
| Q8 | File Descriptors, ulimit & /proc | ⚡ Medium | System |
| Q9 | LVM — Logical Volume Management | ⚡ Medium | Storage |
| Q10 | SSH Advanced — Tunnels, Port Forwarding, Multiplexing | ⚡ Medium | Networking |
| Q11 | Shell Scripting — Advanced Patterns & Error Handling | ⚡ Medium | Scripting |
| Q12 | Linux Security — SELinux, Capabilities, sudoers | 🔥 Hard | Security |
| Q13 | Signals — SIGKILL vs SIGTERM, Graceful Shutdown | ⚡ Medium | Process Management |
| Q14 | strace & lsof — Debugging Running Processes | 🔥 Hard | Debugging |
| Q15 | Linux Namespaces & cgroups — How Containers Work | 🔥 Hard | Containers |
| Q16 | Container Networking — veth, bridge, CNI | 🔥 Hard | K8s Networking |
| Q17 | Linux Kernel Capabilities in Containers | 🔥 Hard | K8s Security |
| Q18 | Node Performance Tuning for Kubernetes | 🔥 Hard | K8s Performance |
| Q19 | eBPF — What It Is and Why It Matters for DevOps | 🔥 Hard | Observability |
| Q20 | Troubleshooting: 5 Classic Production Linux Scenarios | 🔥 Hard | Debugging |

---

---

# Q1 — Linux Process States, Zombies & Orphans

> ⚡ Medium | Process Management

## The Question
**"What are the different process states in Linux? What is a zombie process? What is an orphan process? How do you handle them?"**

## The Answer

Every Linux process exists in one of these states at any moment:

| State | Symbol in `ps` | Meaning |
|-------|---------------|---------|
| **Running** | R | Actively executing on CPU or in run queue |
| **Sleeping (Interruptible)** | S | Waiting for an event (I/O, signal) — can be woken by signal |
| **Sleeping (Uninterruptible)** | D | Waiting on hardware I/O — **cannot** be killed; must wait |
| **Zombie** | Z | Process has exited but parent hasn't read its exit status |
| **Stopped** | T | Paused (Ctrl+Z or SIGSTOP) |
| **Traced** | t | Stopped by a debugger (gdb, strace) |

### Zombie Processes

A zombie is a process that has **finished executing** but still has an entry in the process table because its parent hasn't called `wait()` to read its exit code. The process is dead — it uses no CPU or memory — but it holds a PID.

```bash
# See zombie processes
ps aux | grep Z
# or
ps aux | awk '$8 == "Z" {print}'

# Show the zombie and its parent
ps -ef | grep defunct
```

Why zombies are a problem: Each zombie occupies a PID slot. Linux has a finite PID limit (default 32768). If a buggy parent creates thousands of zombie children, the system can run out of PIDs — no new processes can be spawned, not even a shell to fix the problem.

**How to clear a zombie:**

```bash
# You CANNOT kill a zombie — it's already dead
kill -9 <zombie_pid>   # Does nothing

# You must either:
# 1. Fix the parent to call wait() (code fix)

# 2. Kill the PARENT — init/systemd then reaps the zombie
kill -9 <parent_pid>

# 3. Send SIGCHLD to parent (asks parent to check for dead children)
kill -SIGCHLD <parent_pid>
```

### Orphan Processes

An orphan is a process whose **parent has died** while the child is still running. Linux automatically re-parents orphans to **PID 1 (systemd/init)**, which properly calls `wait()` on them. So orphans are generally harmless.

```bash
# Demonstrate: run a child that outlives its parent
(sleep 100 &) &   # grandparent creates child (sleep) and exits
ps -ef | grep sleep
# PPID will be 1 (reparented to systemd)
```

### The D State (Uninterruptible Sleep) — the Dangerous One

Processes stuck in `D` state cannot be killed with `kill -9`. They're usually waiting on:
- NFS mounts that became unreachable
- Broken disk I/O
- Kernel bug

```bash
# Find D-state processes
ps aux | awk '$8 == "D" {print}'

# What I/O is the process waiting on?
cat /proc/<PID>/wchan    # Shows kernel function where process is blocked
# common output: "nfs_sync_mapping_range" (NFS issue)
# or: "jbd2_journal_commit_transaction" (disk I/O stall)

# Check I/O wait at system level
iostat -x 1 5    # Look for high %await on specific device
```

**Fix for D-state:** Resolve the underlying I/O issue. Unmount a stuck NFS share (`umount -f -l`), fix the disk, or reboot if nothing else works.

---

---

# Q2 — File Permissions — SUID, SGID, Sticky Bit

> ⚡ Medium | Security

## The Question
**"Explain Linux file permissions. What are SUID, SGID, and the sticky bit? Give real examples of each and the security implications."**

## The Answer

### Standard Permissions

```
-rwxr-xr--  1 alice devops 4096 Jan 15 10:00 script.sh
 │││├──┤├──┤
 ││││  │└── Others: r-- (read only)
 ││││  └─── Group:  r-x (read + execute)
 │││└─────── Owner:  rwx (read + write + execute)
 ││└──────── File type: - (regular), d (dir), l (symlink)
```

```bash
# Numeric representation
chmod 754 script.sh   # 7=rwx(owner), 5=r-x(group), 4=r--(others)

# Symbolic
chmod u+x,g-w,o-r script.sh
```

### SUID (Set User ID) — bit 4 in the thousands place

When set on an **executable**, the program runs as the **file's owner**, not the user running it.

```bash
# Example: /usr/bin/passwd
ls -la /usr/bin/passwd
# -rwsr-xr-x 1 root root 68208 /usr/bin/passwd
#     ↑ 's' in owner execute position = SUID

# Any user can change their password, even though /etc/shadow is owned by root
# passwd runs as root (via SUID) to write to /etc/shadow
```

**Set SUID:**
```bash
chmod u+s /path/to/binary
# or
chmod 4755 /path/to/binary    # 4 = SUID prefix
```

**Security implication:** SUID binaries are high-value targets. A vulnerability in a SUID binary can be exploited to gain root access. Audit regularly:

```bash
# Find ALL SUID binaries on the system
find / -perm -4000 -type f 2>/dev/null

# Compare against known baseline — unexpected SUID binaries = red flag
```

### SGID (Set Group ID) — bit 2 in the thousands place

On an **executable**: program runs as the file's group (not user's primary group).

On a **directory**: new files created inside inherit the directory's group — extremely useful for shared project directories.

```bash
# Shared project directory example
mkdir /projects/team-alpha
chgrp alpha /projects/team-alpha
chmod g+s /projects/team-alpha    # Set SGID on directory

# Now any file created in /projects/team-alpha inherits group 'alpha'
# Without SGID: new files get the creating user's primary group
# With SGID: new files always get 'alpha' group — team sharing works
```

```bash
# Find SGID binaries
find / -perm -2000 -type f 2>/dev/null
```

### Sticky Bit — bit 1 in the thousands place

On a **directory**: only the **file's owner** can delete or rename a file, even if the directory is world-writable.

```bash
# Classic example: /tmp
ls -la / | grep tmp
# drwxrwxrwt 1 root root 4096 /tmp
#          ↑ 't' = sticky bit

# Without sticky bit on /tmp: any user could delete any other user's files
# With sticky bit: you can only delete YOUR OWN files in /tmp
```

```bash
chmod +t /shared/uploads
# or
chmod 1777 /shared/uploads
```

### Quick Reference

| Bit | On File | On Directory |
|-----|---------|-------------|
| **SUID (4000)** | Runs as file owner | Ignored (mostly) |
| **SGID (2000)** | Runs as file's group | New files inherit directory's group |
| **Sticky (1000)** | Ignored | Only owner can delete their files |

---

---

# Q3 — Systemd Deep Dive — Units, Targets, Journald

> ⚡ Medium | Init System

## The Question
**"Explain systemd's unit system. What are the unit types? How do you write a service unit file? How does systemd handle dependencies? How do you use journald to debug a failing service?"**

## The Answer

### What systemd replaced

Before systemd (SysV init): services started sequentially via shell scripts in `/etc/init.d/`. Boot was slow because each script waited for the previous to finish. Dependencies were implicit and fragile.

Systemd: parallel startup, declarative dependencies, unified logging (journald), and socket activation.

### Unit Types

| Type | Extension | Purpose |
|------|-----------|---------|
| **Service** | `.service` | Background daemon (nginx, MySQL, your app) |
| **Socket** | `.socket` | Network/IPC socket — activates service on connection |
| **Timer** | `.timer` | Cron replacement — triggers service on schedule |
| **Mount** | `.mount` | Mounts a filesystem |
| **Target** | `.target` | Group of units — like runlevels (`multi-user.target`) |
| **Path** | `.path` | Activates service when file/directory changes |
| **Slice** | `.slice` | cgroup hierarchy for resource management |

### Writing a Service Unit File

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My Application Server
Documentation=https://docs.myapp.com
# Start after network is up AND database is ready
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service     # If postgres fails, this fails too

[Service]
Type=simple                     # Process stays in foreground (most apps)
# Type=forking                  # Process forks to background (traditional daemons)
# Type=notify                   # Process sends sd_notify() when ready (systemd-aware)
# Type=oneshot                  # Run once and exit (good for scripts)

User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp

# Environment
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env  # Load from file (secrets)

# Start command
ExecStart=/usr/bin/node /opt/myapp/server.js

# Pre-start health check (optional)
ExecStartPre=/usr/bin/node /opt/myapp/preflight.js

# Graceful reload (e.g., nginx -s reload)
ExecReload=/bin/kill -HUP $MAINPID

# Restart behavior
Restart=on-failure              # Restart if exits non-zero or killed
RestartSec=5                    # Wait 5 seconds before restart
StartLimitIntervalSec=60        # In 60 seconds...
StartLimitBurst=5               # ...allow max 5 restart attempts, then give up

# Security hardening (best practice)
NoNewPrivileges=true            # Process cannot gain new privileges
ProtectSystem=strict            # /usr, /boot, /etc are read-only
ProtectHome=true                # /home, /root, /run/user are inaccessible
PrivateTmp=true                 # Private /tmp namespace

# Resource limits
LimitNOFILE=65536               # File descriptor limit (override ulimit)
MemoryMax=512M                  # cgroup memory limit

[Install]
WantedBy=multi-user.target      # Enable this service for normal boot
```

### Managing Services

```bash
# Enable and start
sudo systemctl daemon-reload          # After editing unit files
sudo systemctl enable --now myapp     # Enable at boot + start immediately

# Control
sudo systemctl start|stop|restart|reload myapp

# Status and details
systemctl status myapp                # Summary + last log lines
systemctl show myapp                  # ALL properties of the unit

# Dependencies — what would start/stop with this?
systemctl list-dependencies myapp
systemctl list-dependencies --reverse myapp   # Who depends on myapp?

# Targets (runlevels)
systemctl get-default                 # Current default target
sudo systemctl set-default multi-user.target
sudo systemctl isolate rescue.target  # Drop to single-user mode
```

### Journald — Querying Logs

journald is systemd's logging daemon. It captures stdout/stderr of all services + kernel messages.

```bash
# Follow a service in real time
journalctl -u myapp -f

# Last 100 lines
journalctl -u myapp -n 100

# Since a specific time
journalctl -u myapp --since "2025-01-15 14:00" --until "2025-01-15 15:00"
journalctl -u myapp --since "1 hour ago"

# Only errors and above
journalctl -u myapp -p err

# Logs from previous boot (e.g., service that crashed on startup)
journalctl -u myapp -b -1       # -b -1 = previous boot, -b -2 = two boots ago

# Kernel messages only
journalctl -k
journalctl -k | grep -i oom     # Find OOM kills

# All logs since last boot (like /var/log/messages)
journalctl -b

# Output as JSON (for piping to jq)
journalctl -u myapp -o json | jq '.MESSAGE'

# Disk usage by journald
journalctl --disk-usage
# Vacuum old logs
sudo journalctl --vacuum-size=500M
sudo journalctl --vacuum-time=30d
```

### Systemd Timers (Cron Replacement)

```ini
# /etc/systemd/system/log-backup.service
[Unit]
Description=Backup logs to S3

[Service]
Type=oneshot
ExecStart=/opt/scripts/log-to-s3.sh

# /etc/systemd/system/log-backup.timer
[Unit]
Description=Run log-backup every hour
Requires=log-backup.service

[Timer]
OnCalendar=hourly               # Every hour
# OnCalendar=*-*-* 02:00:00    # Daily at 2 AM
# OnBootSec=5min                # 5 minutes after boot
Persistent=true                 # Run immediately if missed (e.g., system was off)

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now log-backup.timer
systemctl list-timers           # Show all timers + next run time
```

Advantage of systemd timers over cron: logs go to journald, can depend on other units, precise timing, `Persistent=true` catches up missed runs.

---

---

# Q4 — Linux Performance Troubleshooting — USE Method

> 🔥 Hard | Performance

## The Question
**"A Linux server is running slowly. Walk me through your performance troubleshooting methodology. What tools do you use and in what order?"**

## The Answer

### The USE Method (Brendan Gregg)

For every resource, check: **Utilization, Saturation, Errors**

| Resource | Utilization | Saturation | Errors |
|----------|------------|------------|--------|
| CPU | `mpstat`, `top` | Run queue length `vmstat r` | `dmesg` |
| Memory | `free -m` | Swap usage, paging `vmstat si/so` | OOM kills `dmesg` |
| Disk | `iostat %util` | I/O queue depth `iostat avgqu-sz` | `dmesg`, `smartctl` |
| Network | `sar -n DEV` | Interface drops/errors `ip -s link` | `netstat -s` |

### Step-by-Step Investigation

**Step 1: 60-second overview — run these first**

```bash
# 1. Load average + uptime (is load high and for how long?)
uptime
# 14:23:01 up 10 days, load average: 8.52, 6.21, 4.18
#                                    1min  5min  15min
# Load > number of CPUs = system is saturated

# 2. How many CPUs? (context for load average)
nproc           # Logical CPUs
lscpu | grep "CPU(s):"

# 3. Are processes in D state (uninterruptible I/O wait)?
ps aux | awk '$8 ~ /D/ {print}'

# 4. Memory: is system paging?
free -m

# 5. Is disk I/O the bottleneck?
iostat -xz 1 5

# 6. Network interface errors/drops?
ip -s link show eth0

# 7. Any kernel errors?
dmesg | tail -20
```

**Step 2: CPU Analysis**

```bash
# Per-CPU utilization breakdown
mpstat -P ALL 1 5
# %usr  %sys  %iowait  %steal  %idle
# High %iowait = process waiting on I/O (disk/network)
# High %steal  = another VM stealing your CPU (noisy neighbour)
# High %sys    = kernel spending time in syscalls

# Which processes are consuming CPU?
top -bn1 | head -20
# Sort by CPU: press P in interactive top
# Sort by Memory: press M

# CPU flame graph (requires perf)
sudo perf record -ag -- sleep 30
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg

# Is a single thread maxing one core? (multi-threaded app bug)
ps -eLf | awk '{print $3, $4}' | sort -k1 -rn | head -10
```

**Step 3: Memory Analysis**

```bash
# Overview
free -m
#              total    used    free   shared  buff/cache  available
# Mem:         15872   14000     200      100        1672        800
# Swap:         4096    3500     596

# Detailed virtual memory stats (run multiple times)
vmstat 1 10
# r  = processes waiting for CPU (run queue)
# b  = processes in uninterruptible sleep
# si = pages swapped IN from disk per second  ← if > 0, you're paging
# so = pages swapped OUT to disk per second   ← if > 0, you're paging

# Which processes are using the most memory?
ps aux --sort=-%mem | head -20

# Where is memory going? (slab caches, page cache etc.)
cat /proc/meminfo | grep -E "MemFree|MemAvailable|Cached|Slab|SwapCached"

# Has OOM killer fired?
dmesg | grep -i "oom\|killed process"
# or
journalctl -k | grep -i oom
```

**Step 4: Disk I/O Analysis**

```bash
# I/O statistics per device
iostat -xz 1 5
# Key metrics:
# %util    = % of time device was busy (>80% = saturated)
# await    = average time for I/O request (ms) — includes queue wait
# r_await  = read latency
# w_await  = write latency
# avgqu-sz = average I/O queue size (>1 = device can't keep up)

# Which process is generating I/O?
iotop -o      # Shows only processes doing I/O
# or
pidstat -d 1  # I/O per process per second

# What files are being read/written?
lsof +D /data    # Open files in /data directory
```

**Step 5: Network Analysis**

```bash
# Overall network throughput
sar -n DEV 1 5       # Bytes/sec, packets/sec per interface
# or
nethogs eth0         # Per-process bandwidth

# TCP connection states
ss -s              # Summary: established, time-wait, etc.
ss -tnp            # All TCP connections with process
ss -tnp state time-wait | wc -l   # Too many TIME_WAIT = connection churn

# Packet drops and errors
ip -s link show eth0
# Errors and drops on RX/TX = hardware/driver issue

# Network latency to a target
ping -c 100 10.0.1.5 | tail -3    # Packet loss and latency stats
mtr --report 10.0.1.5             # Traceroute with per-hop latency
```

**Step 6: Advanced — Latency with `perf` and `BPF`**

```bash
# Track all syscalls for a process
strace -p <PID> -c      # Count mode: shows most-called syscalls + time

# Which syscalls are slowest?
strace -p <PID> -T 2>&1 | awk '{print $NF}' | sort -rn | head

# Block I/O latency histogram (requires bpf-tools)
sudo biolatency-bpfcc 10    # Distribution of disk I/O latency
sudo runqlat-bpfcc 10       # CPU run queue latency
```

### The 5 Most Common Root Causes

```
Problem: High load but low CPU %util
→ Cause: I/O wait (many processes in D state)
→ Check: iostat %iowait, iotop

Problem: App slow but CPU and I/O look fine
→ Cause: Network latency or TCP retransmits
→ Check: ss -s, netstat -s | grep retrans, ping

Problem: Memory looks fine but app crashes
→ Cause: OOM killer firing on cgroup limit (not system RAM)
→ Check: dmesg | grep oom, systemctl show <service> | grep Memory

Problem: High CPU steal
→ Cause: VM host oversubscribed (noisy neighbour in cloud)
→ Check: mpstat %steal > 5% = contact cloud provider

Problem: Slow disk but %util < 50%
→ Cause: High seek time (rotational disk) or small random I/O
→ Check: iostat await (>10ms for random I/O = problem)
```

---

---

# Q5 — Linux Memory — Virtual Memory, OOM Killer, Swap

> 🔥 Hard | Memory

## The Question
**"Explain how Linux virtual memory works. What is the OOM killer and how does it decide what to kill? When does swap help and when does it hurt? How do you tune memory behaviour?"**

## The Answer

### Virtual Memory — The Abstraction

Every process thinks it has its own private address space (virtual addresses). The kernel + MMU (Memory Management Unit) maps virtual addresses to physical RAM pages. This enables:
- Isolation (process A can't read process B's memory)
- Overcommit (promise more memory than physically exists)
- Demand paging (only load pages that are actually accessed)

```
Process sees:          0x0000 ─────── 0xFFFF  (virtual address space)
                                 ↓ (page table translation)
Physical RAM:          actual pages scattered in physical memory
```

### Memory Overcommit

By default, Linux **overcommits** — it allows processes to `malloc()` more memory than physically exists, betting that not all of it will be used simultaneously.

```bash
# Check overcommit settings
cat /proc/sys/vm/overcommit_memory
# 0 = heuristic overcommit (default) — allow most, reject obvious overclaims
# 1 = always overcommit — never deny malloc() (dangerous)
# 2 = never overcommit — strict: total virtual memory ≤ physical + swap

cat /proc/sys/vm/overcommit_ratio
# 50 = with mode 2, virtual memory ≤ RAM + swap + 50% of RAM
```

When the system actually runs out of physical pages to back virtual memory: **OOM killer activates**.

### The OOM Killer

The OOM (Out Of Memory) killer is the kernel's last resort: select a process and kill it to free memory.

**How it selects the victim — OOM score:**

```bash
# Each process has an oom_score (0-1000)
cat /proc/<PID>/oom_score

# Higher score = more likely to be killed
# Score factors:
# - Resident memory usage (bigger = higher score)
# - How long has the process been running (long-running = lower score)
# - Whether it's privileged (root/kernel thread = lower score)

# See all processes sorted by OOM score
for pid in $(ls /proc | grep '^[0-9]'); do
    score=$(cat /proc/$pid/oom_score 2>/dev/null)
    comm=$(cat /proc/$pid/comm 2>/dev/null)
    echo "$score $pid $comm"
done | sort -rn | head -20
```

**Adjusting OOM priority:**

```bash
# oom_score_adj: -1000 to +1000
# -1000 = never kill this process
# +1000 = kill this first

# Protect a critical process (e.g., sshd — you don't want to lose access)
echo -1000 > /proc/$(pidof sshd)/oom_score_adj

# Make a sacrificial process killable first (e.g., a batch job)
echo 500 > /proc/$(pidof batch-job)/oom_score_adj

# In systemd service unit:
# OOMScoreAdjust=-500
```

### Swap — When It Helps vs Hurts

**When swap helps:**
- Provides emergency memory for processes that allocate but rarely use memory
- Allows infrequently-used pages to be moved to disk, freeing RAM for active data
- Prevents OOM killer from firing on brief memory spikes

**When swap hurts:**
- If the system is constantly swapping (paging in/out), throughput collapses
- SSD swap is better than HDD but still orders of magnitude slower than RAM
- In Kubernetes, swap can cause unpredictable pod performance (see Q18)

```bash
# Is the system actively swapping? (si/so columns)
vmstat 1 10
# si = swap in  (reading from swap = was under memory pressure)
# so = swap out (writing to swap = currently under memory pressure)
# Sustained si/so > 0 = add RAM or reduce memory usage

# Swappiness: how aggressively kernel moves pages to swap
cat /proc/sys/vm/swappiness
# 60 = default (fairly aggressive)
# 10 = prefer to keep pages in RAM, only swap when necessary
# 0  = avoid swap completely (OOM killer fires sooner)

# Change temporarily
sudo sysctl vm.swappiness=10

# Persist across reboots
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.d/99-memory.conf
sudo sysctl --system

# For Kubernetes nodes: swappiness=0 (or disable swap entirely)
```

### Page Cache and Buffer Cache

Linux uses free RAM as **page cache** — caching frequently accessed file contents. This is the `buff/cache` in `free -m` output.

```bash
free -m
#               total   used   free  shared  buff/cache  available
# Mem:          15872  12000   1200     200        2672       3200
#                                                 ↑
#          This is not "wasted" RAM — it's file cache
#          "available" = what apps can actually use (free + reclaim cache)
```

The kernel reclaims page cache when a process needs RAM. So `available` is the real usable memory, not `free`.

```bash
# If you genuinely need to free page cache (e.g., before benchmarking)
sudo sync                            # Flush writes
echo 3 | sudo tee /proc/sys/vm/drop_caches   # Drop pagecache + dentries + inodes
# ⚠️ Don't do this in production — page cache is your I/O performance buffer
```

### Transparent Huge Pages (THP)

```bash
# THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# For databases (MySQL, PostgreSQL, MongoDB, Redis): DISABLE THP
# THP causes latency spikes during compaction
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

# Make persistent (add to rc.local or a systemd service)
```

---

---

# Q6 — Advanced Networking — iptables, ss, tcpdump

> 🔥 Hard | Networking

## The Question
**"How does iptables work? What is connection tracking? How do you debug network connectivity issues on a Linux server? Walk through a scenario where traffic is being dropped."**

## The Answer

### iptables Architecture

iptables processes packets through **tables** → **chains** → **rules**. The most important table is `filter`.

```
Packet arrives
      ↓
  PREROUTING (nat table — DNAT: change destination)
      ↓
  Routing decision: for this machine or forward?
  ↙                              ↘
INPUT (local delivery)        FORWARD (transit)
  ↓                              ↓
[Process]                  POSTROUTING (nat table — SNAT/MASQUERADE)
  ↓
OUTPUT (outbound from local process)
  ↓
POSTROUTING
```

**Key chains in the filter table:**
- `INPUT` — packets destined for the local machine
- `OUTPUT` — packets from local processes
- `FORWARD` — packets being routed through (Kubernetes uses this heavily)

```bash
# View current rules with packet counts
sudo iptables -nvL --line-numbers

# View NAT rules (important for Kubernetes)
sudo iptables -t nat -nvL --line-numbers

# View specific chain
sudo iptables -nvL INPUT

# Add a rule: allow port 8080 from specific IP
sudo iptables -A INPUT -s 10.0.1.5 -p tcp --dport 8080 -j ACCEPT

# Block all traffic from an IP
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# Log packets before dropping (for debugging)
sudo iptables -A INPUT -s 10.0.0.0/8 -j LOG --log-prefix "DROPPED: " --log-level 4
sudo iptables -A INPUT -s 10.0.0.0/8 -j DROP
# Logs appear in: journalctl -k | grep "DROPPED:"

# Delete a rule by line number
sudo iptables -D INPUT 3

# Flush all rules (careful in production!)
sudo iptables -F
```

### Connection Tracking (conntrack)

iptables is **stateful** — it tracks connection state:

```bash
# View active connection tracking table
sudo conntrack -L
# tcp      6  431997 ESTABLISHED src=10.0.1.5 dst=10.0.1.10 sport=43210 dport=8080

# States: NEW, ESTABLISHED, RELATED, INVALID
# ESTABLISHED rule (most important — allows return traffic)
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Check conntrack table size (can fill up under DDoS)
cat /proc/sys/net/netfilter/nf_conntrack_count   # Current
cat /proc/sys/net/netfilter/nf_conntrack_max     # Limit
# If count approaches max: SYN packets start dropping silently
```

### Debugging Network Connectivity with `ss`

`ss` is the modern replacement for `netstat`:

```bash
# All listening ports with process names
ss -tlnp
# State    Recv-Q  Send-Q  Local Address:Port  Process
# LISTEN   0       128     0.0.0.0:8080        users:(("node",pid=1234))

# All established connections
ss -tnp state established

# Connections to a specific port
ss -tnp 'dport = :5432'    # Who's connecting to PostgreSQL?

# Connections in TIME_WAIT (too many = connection exhaustion)
ss -tn state time-wait | wc -l

# Send buffer / receive buffer sizes
ss -tmn    # -m for memory info

# Investigate a specific port
ss -tlnp | grep :80
# If nothing here: app isn't listening → check app config, not firewall
```

### tcpdump — The Ground Truth

When you don't know if packets are arriving, `tcpdump` shows the actual packets:

```bash
# Capture all traffic on eth0
sudo tcpdump -i eth0

# Filter by host and port
sudo tcpdump -i eth0 host 10.0.1.5 and port 8080

# Watch only SYN packets (new connections)
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Look for RST packets (connection refused / forcibly closed)
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'

# Write to file for later analysis in Wireshark
sudo tcpdump -i eth0 -w /tmp/capture.pcap host 10.0.1.5

# Verbose output — show full headers
sudo tcpdump -i eth0 -vvv port 8080

# Look at DNS queries
sudo tcpdump -i eth0 port 53

# Capture on all interfaces
sudo tcpdump -i any port 8080
```

### Debugging Scenario: Traffic Being Dropped

**Symptom:** Client gets "Connection refused" or connection times out to port 8080 on server.

```bash
# Step 1: Is the app even listening?
ss -tlnp | grep :8080
# If nothing: app isn't running or listening on wrong interface

# Step 2: Is it listening on 127.0.0.1 instead of 0.0.0.0?
ss -tlnp | grep :8080
# LISTEN 0 128 127.0.0.1:8080  ← bound to localhost only!
# Fix: change app config to bind to 0.0.0.0 or specific interface IP

# Step 3: Is iptables blocking it?
sudo iptables -nvL INPUT | grep 8080
# If DROP rule matches: add ACCEPT rule before it

# Step 4: Are packets even arriving?
sudo tcpdump -i eth0 port 8080
# If SYN packets arriving but no SYN-ACK: OS or app is dropping them
# If no SYN packets: problem is upstream (routing, security group)

# Step 5: Check AWS security groups (if on EC2)
aws ec2 describe-security-groups --group-ids sg-xxx \
  | jq '.SecurityGroups[].IpPermissions'

# Step 6: Check system-level port limits
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768 60999  (ephemeral port range)
# If exhausted: ss -s | grep "estab" — too many connections
```

---

---

# Q7 — Linux Boot Process — BIOS to Userspace

> ⚡ Medium | System

## The Question
**"Walk me through what happens from the moment you press the power button on a Linux server to the point where a user can log in."**

## The Answer

```
Power on
   ↓
BIOS/UEFI firmware
   ↓
Bootloader (GRUB2)
   ↓
Linux Kernel
   ↓
initramfs (temporary root filesystem)
   ↓
Real root filesystem mounted
   ↓
systemd (PID 1)
   ↓
Target units (multi-user.target)
   ↓
Login prompt
```

### Stage 1: BIOS / UEFI

- **BIOS (Legacy):** POST (Power-On Self Test) → loads first 512 bytes (MBR) from boot disk → hands off to bootloader
- **UEFI (Modern):** More capable firmware → reads EFI System Partition (FAT32 partition at `/boot/efi`) → loads bootloader as a `.efi` file

```bash
# Check if system is UEFI or BIOS
ls /sys/firmware/efi 2>/dev/null && echo UEFI || echo BIOS
```

### Stage 2: GRUB2

GRUB (Grand Unified Bootloader) presents the OS selection menu, loads the kernel and initramfs into RAM, and hands control to the kernel.

```bash
# GRUB config
cat /boot/grub2/grub.cfg          # Generated file — don't edit directly
cat /etc/default/grub             # Edit this, then regenerate
sudo grub2-mkconfig -o /boot/grub2/grub.cfg    # Regenerate

# Boot to a specific kernel (useful after bad kernel update)
# In GRUB menu: press 'e' to edit, change the kernel line, Ctrl+X to boot

# GRUB rescue mode: if GRUB itself is broken
# grub> ls                        # List partitions
# grub> ls (hd0,1)/               # List files on partition
# grub> set root=(hd0,1)
# grub> linux /boot/vmlinuz-...   root=/dev/sda1
# grub> initrd /boot/initramfs-...
# grub> boot
```

### Stage 3: Linux Kernel Initialization

- Kernel decompresses itself
- Initializes CPU, memory management, hardware drivers
- Mounts **initramfs** (initial RAM filesystem) — a temporary root FS in RAM containing minimal tools needed to mount the real root filesystem
- Runs `/init` (or `/sbin/init`) inside initramfs

```bash
# Kernel parameters (passed by GRUB)
cat /proc/cmdline
# BOOT_IMAGE=/boot/vmlinuz-5.15.0 root=/dev/nvme0n1p1 ro quiet

# Common troubleshooting kernel params:
# rd.break      = drop to shell inside initramfs (before pivot_root)
# init=/bin/sh  = skip systemd, go straight to shell
# systemd.unit=rescue.target = boot to rescue mode
```

### Stage 4: systemd Takes Over

Once the real root filesystem is mounted, the kernel runs `/sbin/init` → which is symlinked to `systemd`.

```bash
ls -la /sbin/init
# /sbin/init -> /lib/systemd/systemd

# systemd reads its configuration and starts units in dependency order
# Default target is usually multi-user.target (text) or graphical.target

# Boot time analysis
systemd-analyze                    # Total boot time
systemd-analyze blame              # Time each unit took to start
systemd-analyze critical-chain     # The critical path (slowest chain)
systemd-analyze plot > boot.svg    # Visual boot timeline

# Example output:
# Startup finished in 1.2s (kernel) + 3.4s (initrd) + 8.6s (userspace) = 13.2s
```

### Troubleshooting Boot Failures

```bash
# Can't boot? Use rescue/emergency mode
# Add to GRUB kernel line: systemd.unit=emergency.target
# You get a root shell with minimal services

# Check failed units after boot
systemctl --failed

# View boot log
journalctl -b              # Current boot
journalctl -b -1           # Previous boot (if system crashed)
journalctl -b -p err       # Only errors from current boot
```

---

---

# Q8 — File Descriptors, ulimit & /proc

> ⚡ Medium | System

## The Question
**"What are file descriptors? What happens when a process runs out of them? How do you diagnose and fix 'too many open files' errors in production?"**

## The Answer

### What Are File Descriptors?

Every open file, socket, pipe, or device is represented by a **file descriptor (FD)** — an integer index into the process's open file table.

```
FD 0 → stdin
FD 1 → stdout
FD 2 → stderr
FD 3 → /var/log/app.log
FD 4 → socket: 10.0.1.5:43210 → database:5432
FD 5 → /tmp/lock.pid
...
FD N → socket: 10.0.1.100:50123 → user connection
```

Every HTTP connection to a web server is a file descriptor. A busy server handling 10,000 concurrent connections needs 10,000+ file descriptors just for those connections (plus log files, config files, etc.).

### The Limits

```bash
# System-wide maximum (all processes combined)
cat /proc/sys/fs/file-max
# 9223372036854775807 (effectively unlimited on modern kernels)

# Per-process limits (soft and hard)
ulimit -n         # Soft limit (current process) — default 1024
ulimit -Hn        # Hard limit (ceiling for soft limit) — default 1048576

# Check limits for a RUNNING process
cat /proc/<PID>/limits | grep "open files"
# Max open files   65536   65536   files

# Check how many FDs a process currently has open
ls /proc/<PID>/fd | wc -l
# or
lsof -p <PID> | wc -l
```

### Diagnosing "Too Many Open Files"

```bash
# Error in app logs: "java.io.IOException: Too many open files"
# or: "EMFILE: too many open files"

# Step 1: Find the process hitting the limit
lsof 2>/dev/null | awk '{print $2}' | sort | uniq -c | sort -rn | head -10
# Count open files per PID

# Step 2: What files does that process have open?
lsof -p <PID>
# Look for: sockets in CLOSE_WAIT (app not closing connections)
# Look for: open log files (not being rotated/closed)

# Step 3: Are sockets leaking?
lsof -p <PID> | grep -c "IPv4\|IPv6"   # Count open sockets

# Step 4: Check TCP connections in CLOSE_WAIT (app-side bug)
ss -tnp | grep CLOSE_WAIT
# CLOSE_WAIT means: remote end closed, local app hasn't called close()
# Too many CLOSE_WAIT = connection leak in application code
```

### Fixing the Limits

```bash
# Temporary: raise for current shell session
ulimit -n 65536

# Permanent: per-user/group limits
sudo vim /etc/security/limits.conf
# appuser soft nofile 65536
# appuser hard nofile 65536
# * hard nofile 1048576    # All users

# For systemd services (best practice for production services):
# Add to [Service] section of the .service unit file:
LimitNOFILE=65536

# System-wide (applies to all new processes):
echo "fs.file-max = 2097152" | sudo tee -a /etc/sysctl.d/99-files.conf
sudo sysctl --system
```

### /proc — The Virtual Filesystem

`/proc` is a virtual filesystem that exposes kernel data structures. It's how most monitoring tools work internally.

```bash
/proc/meminfo           # Memory statistics (free, buffers, cache)
/proc/cpuinfo           # CPU details (cores, speed, flags)
/proc/loadavg           # Load average
/proc/net/tcp           # All TCP connections (raw)
/proc/sys/              # Tunable kernel parameters
/proc/<PID>/            # Per-process information
  /proc/<PID>/cmdline   # Full command line
  /proc/<PID>/environ   # Environment variables
  /proc/<PID>/fd/       # Symbolic links to open file descriptors
  /proc/<PID>/maps      # Memory map (what's loaded where)
  /proc/<PID>/status    # Process state, memory usage
  /proc/<PID>/net/tcp   # This process's network connections
```

```bash
# See exactly what a process has open
ls -la /proc/1234/fd
# lrwx------ 1 app app 64 fd/3 -> /var/log/app.log
# lrwx------ 1 app app 64 fd/4 -> socket:[12345]

# Recover contents of a deleted-but-open file
# (classic trick when someone accidentally deletes a log file)
cat /proc/<PID>/fd/3 > /tmp/recovered.log
```

---

---

# Q9 — LVM — Logical Volume Management

> ⚡ Medium | Storage

## The Question
**"What is LVM and why would you use it over standard partitions? How do you extend a logical volume online? What's the process for adding a new disk?"**

## The Answer

LVM adds an abstraction layer between physical disks and filesystems, enabling: online resize, snapshots, and spanning volumes across multiple physical disks.

```
Physical Disks:     /dev/sdb    /dev/sdc    /dev/sdd
                       ↓           ↓           ↓
Physical Volumes:   PV(sdb)    PV(sdc)    PV(sdd)
                         ↘         ↓        ↙
Volume Group:              VG: data-vg
                          ↙         ↘
Logical Volumes:    LV: app-lv    LV: logs-lv
                        ↓               ↓
Filesystems:     /dev/data-vg/app-lv  /dev/data-vg/logs-lv
                        ↓               ↓
Mount Points:         /app           /var/log
```

### Common LVM Commands

```bash
# Display current setup
pvs             # Physical volumes
vgs             # Volume groups
lvs             # Logical volumes
pvdisplay       # Detailed PV info
vgdisplay       # Detailed VG info
lvdisplay       # Detailed LV info

# Full overview
lsblk
```

### Extending a Logical Volume Online (No Downtime)

**Scenario:** `/app` partition is 80% full. The VG has free space.

```bash
# Check free space in the VG
vgs
# VG        #PV #LV #SN Attr VSize  VFree
# data-vg     2   2   0 wz--n- 200g   40g   ← 40GB free

# Extend the LV by 20GB
sudo lvextend -L +20G /dev/data-vg/app-lv

# OR extend to a specific size
sudo lvextend -L 80G /dev/data-vg/app-lv

# OR use all remaining free space
sudo lvextend -l +100%FREE /dev/data-vg/app-lv

# Extend the FILESYSTEM (no unmount needed for ext4/XFS)
# ext4:
sudo resize2fs /dev/data-vg/app-lv

# XFS:
sudo xfs_growfs /app      # (pass mount point, not device)

# Verify
df -hT /app

# Shortcut: extend LV AND filesystem in one command
sudo lvextend -L +20G -r /dev/data-vg/app-lv
# -r = resize filesystem automatically
```

### Adding a New Physical Disk

**Scenario:** VG has no free space. New disk `/dev/sdd` has been added.

```bash
# Initialize new disk as PV
sudo pvcreate /dev/sdd

# Add PV to the existing VG
sudo vgextend data-vg /dev/sdd

# Confirm new free space
vgs    # VFree should now show the new disk's space

# Now extend LV as above
sudo lvextend -L +50G -r /dev/data-vg/app-lv
```

### LVM Snapshots (for Backup)

```bash
# Create a snapshot of app-lv (requires free space in VG)
sudo lvcreate -L 5G -s -n app-snapshot /dev/data-vg/app-lv

# Mount the snapshot (read-only backup)
sudo mount -o ro /dev/data-vg/app-snapshot /mnt/backup

# Backup from the snapshot
tar -czf /backup/app-$(date +%Y%m%d).tar.gz /mnt/backup/

# Remove snapshot after backup
sudo umount /mnt/backup
sudo lvremove /dev/data-vg/app-snapshot
```

---

---

# Q10 — SSH Advanced — Tunnels, Port Forwarding, Multiplexing

> ⚡ Medium | Networking

## The Question
**"Explain SSH tunneling — local, remote, and dynamic port forwarding. When would you use each? What is SSH multiplexing and why does it speed up deployments?"**

## The Answer

### Local Port Forwarding (-L)

*"I want to access a remote service that isn't publicly reachable."*

```bash
# Forward local port 5433 → remote PostgreSQL at 5432
# (the DB is in a private subnet, only reachable from the bastion)
ssh -L 5433:db-server:5432 user@bastion-host

# Now on your laptop:
psql -h localhost -p 5433 -U admin mydb
# ← Traffic goes: laptop:5433 → bastion → db-server:5432

# Non-interactive (background tunnel)
ssh -fNL 5433:db-server:5432 user@bastion-host
# -f = background, -N = no remote command (tunnel only)
```

**~/.ssh/config version:**
```
Host db-tunnel
  HostName bastion-host
  User ec2-user
  LocalForward 5433 db-server.internal:5432
  ServerAliveInterval 60
```

### Remote Port Forwarding (-R)

*"I want a remote server to reach a service on my local machine."*

```bash
# Expose my local webhook receiver (port 3000) through the remote server
ssh -R 8080:localhost:3000 user@public-server

# Now http://public-server:8080 → your laptop's port 3000
# Useful for: testing webhooks from GitHub/Stripe without deploying
```

### Dynamic Port Forwarding (-D) — SOCKS Proxy

*"I want to route all my traffic through a remote server."*

```bash
# Create a SOCKS5 proxy on local port 1080
ssh -D 1080 user@bastion-host

# Configure browser to use SOCKS5 proxy: localhost:1080
# All browser traffic now routes through the bastion server
# Useful for: accessing an entire private network through one SSH tunnel
```

### SSH Multiplexing — Speed Up Deployments

By default, each `ssh` command opens a new TCP connection + full key exchange (~200-300ms overhead). Multiplexing reuses one connection for multiple SSH sessions.

```
Without multiplexing:
ssh cmd1   → new TCP connection → auth → execute → close   (300ms)
ssh cmd2   → new TCP connection → auth → execute → close   (300ms)
ssh cmd3   → new TCP connection → auth → execute → close   (300ms)
Total: 900ms+ overhead

With multiplexing:
ssh cmd1   → new TCP connection → auth → execute           (300ms first)
ssh cmd2   → reuse connection → execute                    (5ms)
ssh cmd3   → reuse connection → execute                    (5ms)
Total: 310ms overhead
```

```
# ~/.ssh/config — enable multiplexing
Host *
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p   # Socket file for the master connection
  ControlPersist 10m               # Keep master connection open 10min after last use
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

This is a massive win in Ansible, Fabric, or any deployment tool that runs many SSH commands against the same host.

### SSH Config — Full Production Example

```
# ~/.ssh/config

# Default settings for all hosts
Host *
  ServerAliveInterval 60
  ServerAliveCountMax 3
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
  AddKeysToAgent yes

# Bastion
Host bastion-prod
  HostName 54.123.45.67
  User ec2-user
  IdentityFile ~/.ssh/prod-key.pem
  ForwardAgent yes

# Private servers via bastion
Host app-server-*
  User ec2-user
  IdentityFile ~/.ssh/prod-key.pem
  ProxyJump bastion-prod

Host app-server-1
  HostName 10.0.1.10

Host app-server-2
  HostName 10.0.1.11
```

---

---

# Q11 — Shell Scripting — Advanced Patterns & Error Handling

> ⚡ Medium | Scripting

## The Question
**"What are the most important bash scripting best practices for production scripts? How do you handle errors, parse arguments, write reusable functions, and make scripts safe?"**

## The Answer

### The Safety Header — Always Start With This

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e : exit immediately on error
# -u : treat undefined variables as errors (catches typos)
# -o pipefail : catch errors in pipelines (cmd1 | cmd2 — catches cmd1 failing)

# For debugging: print each command before executing
# set -x

# Trap: run cleanup on exit (success or failure)
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/tempfile.$$   # $$ = current PID (unique temp files)
    # remove lock files, kill background processes, etc.
}
trap cleanup EXIT            # Always runs on exit
trap 'echo "Error on line $LINENO"' ERR   # Extra info on errors
```

### Variable Safety

```bash
# BAD: unquoted variables break on spaces and glob patterns
for file in $FILE_LIST; do ...    # Breaks if filenames have spaces
rm $TMPDIR/*                      # Expands glob — dangerous

# GOOD: always quote variables
for file in "$FILE_LIST"; do ...
rm "$TMPDIR"/*

# BAD: variable might be empty → 'rm -rf /'
rm -rf "$TMPDIR/"
# GOOD: check first
[[ -n "$TMPDIR" ]] && rm -rf "$TMPDIR/"
# or use parameter expansion default
rm -rf "${TMPDIR:-/tmp/myapp-$$}/"
```

### Argument Parsing with getopts

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    echo "Usage: $0 [-e environment] [-d] [-h]"
    echo "  -e  environment (dev|staging|prod) [required]"
    echo "  -d  enable dry-run"
    echo "  -h  show this help"
    exit 1
}

ENV=""
DRY_RUN=false

while getopts "e:dh" opt; do
    case $opt in
        e) ENV="$OPTARG" ;;
        d) DRY_RUN=true ;;
        h) usage ;;
        *) usage ;;
    esac
done

# Validate required argument
[[ -z "$ENV" ]] && { echo "ERROR: -e environment is required"; usage; }
[[ "$ENV" =~ ^(dev|staging|prod)$ ]] || { echo "ERROR: invalid environment '$ENV'"; usage; }

echo "Running in environment: $ENV (dry-run: $DRY_RUN)"
```

### Functions and Return Codes

```bash
# Functions return 0 = success, non-zero = failure (like commands)
check_aws_credentials() {
    if ! aws sts get-caller-identity &>/dev/null; then
        echo "ERROR: AWS credentials not configured"
        return 1
    fi
    return 0
}

deploy_service() {
    local service_name="$1"    # local = function-scoped variable
    local image_tag="$2"

    echo "Deploying $service_name with tag $image_tag"
    # ...
}

# Call functions with error checking
check_aws_credentials || exit 1
deploy_service "myapp" "v1.2.3"
```

### Logging Helpers

```bash
# Color-coded logging
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'

log_info()  { echo -e "${GREEN}[INFO]${NC}  $(date +%H:%M:%S) $*"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC}  $(date +%H:%M:%S) $*" >&2; }
log_error() { echo -e "${RED}[ERROR]${NC} $(date +%H:%M:%S) $*" >&2; }

# Usage
log_info "Starting deployment"
log_warn "Config file not found, using defaults"
log_error "Database connection failed"

# Also log to file
exec > >(tee -a /var/log/deploy.log) 2>&1
# All subsequent output goes to both terminal AND logfile
```

### Retry Logic

```bash
# Retry a command N times with backoff
retry() {
    local max_attempts=$1
    local delay=$2
    shift 2
    local cmd=("$@")
    local attempt=1

    until "${cmd[@]}"; do
        if (( attempt >= max_attempts )); then
            log_error "Command failed after $max_attempts attempts: ${cmd[*]}"
            return 1
        fi
        log_warn "Attempt $attempt failed. Retrying in ${delay}s..."
        sleep "$delay"
        (( attempt++ ))
        (( delay *= 2 ))   # Exponential backoff
    done
}

# Usage
retry 5 2 aws s3 cp /var/log/app.log s3://bucket/logs/
retry 3 5 curl -f https://api.example.com/health
```

---

---

# Q12 — Linux Security — SELinux, Capabilities, sudoers

> 🔥 Hard | Security

## The Question
**"What is SELinux and how does it differ from standard Linux permissions? What are Linux capabilities and why do they matter for containers? How do you write a secure sudoers policy?"**

## The Answer

### SELinux (Security-Enhanced Linux)

Standard Linux permissions ask: *"Who are you?"* (UID/GID).
SELinux asks: *"What is this process allowed to do, regardless of who runs it?"*

SELinux enforces **mandatory access control (MAC)** — policies are set by the system administrator and even root cannot override them.

```bash
# Check SELinux status
sestatus
getenforce    # Enforcing | Permissive | Disabled

# Modes:
# Enforcing  = blocks and logs violations
# Permissive = logs violations only (good for testing new policies)
# Disabled   = no SELinux

# Switch to permissive temporarily (troubleshooting)
sudo setenforce 0    # Permissive (resets on reboot)
sudo setenforce 1    # Back to Enforcing

# Permanent mode change
sudo sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

**SELinux labels — everything has a context:**

```bash
# File contexts
ls -Z /var/www/html/index.html
# -rw-r--r--. root root system_u:object_r:httpd_sys_content_t:s0 index.html
#                        ↑user    ↑role    ↑type (most important)  ↑level

# Process contexts
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0  httpd

# httpd_t process can read httpd_sys_content_t files → allowed
# httpd_t process CANNOT read shadow_t files → DENIED even if file perms say 644
```

**Common SELinux troubleshooting:**

```bash
# Why is something being denied?
sudo ausearch -m avc -ts recent | audit2why
# Output: "Missing type enforcement (TE) allow rule"

# View SELinux denials in real time
sudo tail -f /var/log/audit/audit.log | grep denied

# Auto-generate a policy module from denials (careful in prod!)
sudo ausearch -m avc -ts recent | audit2allow -M mypolicy
sudo semodule -i mypolicy.pp    # Install policy

# Fix file context (e.g., after moving files)
sudo restorecon -Rv /var/www/html/
# or manually set context
sudo chcon -t httpd_sys_content_t /var/www/html/myapp/
```

### Linux Capabilities

Traditionally: root = ALL powers, non-root = NO special powers. Binary.

Capabilities break root's powers into discrete units that can be granted individually:

| Capability | What it allows |
|-----------|---------------|
| `CAP_NET_BIND_SERVICE` | Bind to ports < 1024 (without being root) |
| `CAP_NET_RAW` | Create raw sockets (ping, tcpdump) |
| `CAP_SYS_ADMIN` | Mount filesystems, modify kernel params — like a mini-root |
| `CAP_DAC_OVERRIDE` | Bypass file permission checks |
| `CAP_KILL` | Send signals to other users' processes |
| `CAP_SYS_PTRACE` | Debug other processes (strace, gdb) |

```bash
# Check capabilities on a binary
getcap /usr/bin/ping
# /usr/bin/ping = cap_net_raw+ep
# ep = effective + permitted

# Set capability on a binary (so it doesn't need setuid root)
sudo setcap cap_net_bind_service=+ep /usr/bin/node
# Now node can bind to port 80 without being root or SUID

# Get all files with capabilities
getcap -r / 2>/dev/null

# Check capabilities of a running process
cat /proc/<PID>/status | grep Cap
# CapPrm: 00000000a82425fb  (permitted)
# CapEff: 00000000a82425fb  (effective)
# Decode it:
capsh --decode=00000000a82425fb
```

### Secure sudoers Policy

```bash
# NEVER edit /etc/sudoers directly
sudo visudo    # Uses a syntax checker — saves you from locking yourself out

# View current sudoers policies
sudo cat /etc/sudoers
sudo ls /etc/sudoers.d/

# Structure: WHO  WHERE=(AS_WHOM)  COMMAND

# BAD: gives full root (common but wrong)
devuser ALL=(ALL) ALL

# GOOD: specific commands only
devuser ALL=(ALL) NOPASSWD: /bin/systemctl restart myapp, /usr/bin/docker ps

# Group-based (better for teams)
%devops ALL=(ALL) NOPASSWD: /bin/systemctl * myapp

# Restrict to specific hosts
devuser webserver1=(ALL) /bin/systemctl restart nginx

# Log all sudo commands (enabled by default in modern systems)
Defaults logfile=/var/log/sudo.log
Defaults log_input, log_output    # Log what they type AND see

# Require password even for NOPASSWD override
Defaults timestamp_timeout=0    # Always ask for password (no cache)
```

---

---

# Q13 — Signals — SIGKILL vs SIGTERM, Graceful Shutdown

> ⚡ Medium | Process Management

## The Question
**"What is the difference between SIGTERM and SIGKILL? How do you implement graceful shutdown in an application? What signals does Kubernetes send when terminating a pod?"**

## The Answer

### Signal Overview

| Signal | Number | Can be caught? | Default Action | Use Case |
|--------|--------|---------------|----------------|---------|
| **SIGHUP** | 1 | Yes | Terminate | Reload config (nginx, sshd) |
| **SIGINT** | 2 | Yes | Terminate | Ctrl+C — user interruption |
| **SIGQUIT** | 3 | Yes | Core dump | Ctrl+\ — quit with dump |
| **SIGKILL** | 9 | **No** | Kill immediately | Force kill — cannot be ignored |
| **SIGTERM** | 15 | Yes | Terminate | Graceful shutdown request |
| **SIGSTOP** | 19 | **No** | Stop process | Pause — cannot be ignored |
| **SIGCONT** | 18 | Yes | Continue | Resume a stopped process |
| **SIGCHLD** | 17 | Yes | Ignore | Child process changed state |

### SIGTERM vs SIGKILL — The Key Difference

```
SIGTERM (15): "Please shut down gracefully."
              ↓
              Process RECEIVES the signal
              Process can:
              - Finish current requests
              - Flush write buffers
              - Close database connections
              - Clean up temp files
              - Save state
              Then exit cleanly.

SIGKILL  (9): "Die NOW."
              ↓
              Kernel kills the process instantly
              Process has ZERO opportunity to clean up
              Can cause:
              - Corrupt database writes
              - Lost in-flight requests
              - Orphaned lock files
              - Connection pool exhaustion on restart
```

```bash
# Send SIGTERM (graceful) — always try this first
kill <PID>            # Default is SIGTERM
kill -15 <PID>        # Explicit
kill -SIGTERM <PID>   # Same

# Wait for graceful shutdown, then SIGKILL if needed
kill -SIGTERM <PID>
sleep 30
kill -0 <PID> 2>/dev/null && kill -SIGKILL <PID>
# kill -0 = check if process exists without sending a signal

# The correct shutdown sequence (Kubernetes does this):
# 1. SIGTERM → wait grace period → SIGKILL
```

### Implementing Graceful Shutdown (Application Code)

**Node.js:**
```javascript
process.on('SIGTERM', async () => {
    console.log('SIGTERM received. Shutting down gracefully...');
    
    // Stop accepting new connections
    server.close(async () => {
        console.log('HTTP server closed');
        
        // Close database connections
        await db.disconnect();
        
        // Flush pending log writes
        logger.flush();
        
        console.log('Cleanup complete. Exiting.');
        process.exit(0);
    });
    
    // Force exit after 25 seconds (K8s grace period is 30s)
    setTimeout(() => {
        console.error('Forced exit after timeout');
        process.exit(1);
    }, 25000);
});
```

**Python:**
```python
import signal
import sys

def graceful_shutdown(signum, frame):
    print("SIGTERM received, shutting down...")
    server.shutdown()       # Stop accepting new requests
    db_pool.closeall()      # Close DB connections
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
signal.signal(signal.SIGINT, graceful_shutdown)
```

### Kubernetes Pod Termination Sequence

```
kubectl delete pod my-pod
    ↓
1. Pod moves to "Terminating" state
2. K8s removes pod from Service endpoints (traffic stops routing to it)
3. K8s sends SIGTERM to PID 1 in each container
4. Grace period begins (default: 30 seconds)
    ↓ (pod handles SIGTERM and cleans up)
5. After grace period: K8s sends SIGKILL
6. Container is force-killed
7. Pod is removed from etcd
```

```yaml
# Configure grace period in pod spec
spec:
  terminationGracePeriodSeconds: 60   # Give app 60s to clean up (default 30)
  containers:
    - name: myapp
      lifecycle:
        preStop:            # Hook runs BEFORE SIGTERM is sent
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
            # Sleep 5s to give kube-proxy time to update iptables rules
            # Without this, some requests in-flight may hit a dying pod
```

**Important:** If your app is not PID 1 (e.g., it's started by a shell script), SIGTERM goes to the shell, not your app. Use `exec` in shell scripts to replace the shell process:

```bash
#!/bin/bash
# BAD: SIGTERM goes to bash, NOT to node
node server.js

# GOOD: exec replaces bash with node — node becomes PID 1
exec node server.js
```

---

---

# Q14 — strace & lsof — Debugging Running Processes

> 🔥 Hard | Debugging

## The Question
**"How do you use strace and lsof to debug a misbehaving process? Give specific examples of what you can discover with each tool."**

## The Answer

### strace — See Every System Call

A system call (syscall) is how a process asks the kernel to do something: open a file, write data, allocate memory, create a network socket. `strace` shows every syscall a process makes in real time.

```bash
# Attach to a running process
sudo strace -p <PID>

# Count syscalls (find the most frequent ones)
sudo strace -p <PID> -c
# % time  seconds  usecs/call  calls   errors  syscall
# 45.12   0.045123      45    1000        0    futex
# 32.11   0.032000      32    1000        0    read
# ← A process spending 45% of time in futex is likely lock-contended

# Trace a specific syscall only
sudo strace -p <PID> -e trace=open,read,write

# Trace file operations
sudo strace -p <PID> -e trace=file

# Trace network operations
sudo strace -p <PID> -e trace=network

# Show timing for each call
sudo strace -p <PID> -T
# write(4, "data...", 100)   = 100 <0.000312>
#                                   ↑ 312 microseconds

# Trace a new process (and all its children)
strace -f -e trace=execve ls -la /tmp
```

**What you can discover with strace:**

```bash
# Why is the app slow? (finds slow syscalls)
sudo strace -p <PID> -T 2>&1 | awk '{print $NF}' | sort -rn | head -5

# Why is the app failing? (finds ENOENT, EACCES errors)
sudo strace -p <PID> 2>&1 | grep -E "ENOENT|EACCES|EPERM"
# open("/etc/config.yaml", O_RDONLY)  = -1 ENOENT (No such file or directory)
# → Config file is in the wrong location

# What database queries is the app making? (trace network writes)
sudo strace -p <PID> -e trace=write -s 4096 2>&1 | grep -A1 "write.*5432"

# Is the process stuck waiting for a lock?
sudo strace -p <PID>
# If you see: futex(0x..., FUTEX_WAIT, ...) for a long time
# → Thread is blocked waiting for a mutex — deadlock or lock contention
```

### lsof — Everything a Process Has Open

`lsof` (List Open Files) shows every file descriptor a process has open — files, sockets, pipes, devices.

```bash
# All files open by a process
lsof -p <PID>

# Column meanings:
# COMMAND   PID   USER   FD   TYPE   DEVICE   SIZE   NODE   NAME
# node     1234  app     4u   IPv4  123456    TCP    localhost:46812->db:5432 (ESTABLISHED)
#                        ↑FD number, u=read+write, r=read, w=write
#                              ↑IPv4 socket

# Files open by a specific user
lsof -u appuser

# Processes using a specific file
lsof /var/log/app.log

# Processes using a specific port
lsof -i :8080
lsof -i TCP:8080 -sTCP:LISTEN    # Only listening

# All network connections
lsof -i                    # All
lsof -i TCP                # TCP only
lsof -i @10.0.1.5          # Connections to specific host

# Find deleted files still held open (disk space not freed)
lsof +L1
# The +L1 means: link count < 1 (file deleted from directory but still open)

# Count open FDs per process
lsof 2>/dev/null | awk '{print $2}' | sort | uniq -c | sort -rn | head
```

**Debugging scenarios with lsof:**

```bash
# Scenario: "Port 8080 is already in use" on app start
lsof -i :8080
# node  5678  app  12u  IPv4  99999  TCP *:8080 (LISTEN)
# ← PID 5678 already using port 8080

# Scenario: Disk full but can't find the file
df -hT        # /var/log full
du -sh /var/log/*   # All files look small
# Mystery: deleted file still held open
lsof +L1 | grep /var/log
# node  1234  app  5w  REG  8,1  5368709120  deleted /var/log/app.log
# ↑ 5GB file, deleted but node has it open → truncate or restart node

# Scenario: Database connection pool leak
lsof -p <PID> | grep 5432 | wc -l
# Shows 500 connections to PostgreSQL when pool max is 50
# ← Connection leak — connections opened but never closed
```

---

---

# Q15 — Linux Namespaces & cgroups — How Containers Work

> 🔥 Hard | Containers (K8s)

## The Question
**"Explain how containers work at the Linux kernel level. What are namespaces and cgroups? How do Docker/Kubernetes use them?"**

## The Answer

Containers are **not VMs**. There is no hypervisor, no separate kernel. A container is just a Linux process with:
1. **Namespaces** — isolated view of system resources (process can't see outside its namespace)
2. **cgroups** — limits on resource consumption (CPU, memory, I/O)

### Linux Namespaces

Each namespace type isolates a different aspect of the OS:

| Namespace | Flag | What it isolates |
|-----------|------|-----------------|
| **PID** | `CLONE_NEWPID` | Process IDs — container has its own PID 1 |
| **Network** | `CLONE_NEWNET` | Network interfaces, routing tables, iptables |
| **Mount** | `CLONE_NEWNS` | Filesystem mount points |
| **UTS** | `CLONE_NEWUTS` | Hostname and domain name |
| **IPC** | `CLONE_NEWIPC` | Shared memory, semaphores |
| **User** | `CLONE_NEWUSER` | UID/GID mappings (rootless containers) |
| **Cgroup** | `CLONE_NEWCGROUP` | cgroup root view |

```bash
# See all namespaces on the system
lsns

# See namespaces of a container process
ls -la /proc/<container_PID>/ns/
# lrwxrwxrwx net -> net:[4026532123]   ← unique namespace ID
# lrwxrwxrwx pid -> pid:[4026532456]
# Compare with host PID 1's namespaces:
ls -la /proc/1/ns/
# Different namespace IDs = isolated

# Enter a running container's namespaces (what `docker exec` does)
sudo nsenter -t <container_PID> --pid --net --mount --uts --ipc /bin/bash
# You're now inside the container's view of the world
```

**PID namespace in practice:**

```bash
# In a container:
ps aux
# PID 1 = your application (nginx, node, etc.)
# Container thinks it's the only process

# On the host:
ps aux | grep nginx
# PID 38271 = nginx  ← host sees the real PID
# The container's "PID 1" is actually PID 38271 on the host
```

### cgroups (Control Groups)

cgroups limit and account for resource usage. They're what prevents one container from consuming all CPU/memory on a node.

```bash
# cgroups v2 — modern Linux (Amazon Linux 2023, Ubuntu 22+)
# Hierarchy at /sys/fs/cgroup/

# Find a container's cgroup
cat /proc/<container_PID>/cgroup
# 0::/system.slice/docker-abc123.scope

# Inspect the cgroup limits
cat /sys/fs/cgroup/system.slice/docker-abc123.scope/memory.max
# 536870912  ← 512MB memory limit

cat /sys/fs/cgroup/system.slice/docker-abc123.scope/cpu.max
# 100000 1000000  ← 100ms per 1000ms = 0.1 CPU (100 millicores)

# Real-time cgroup stats
cat /sys/fs/cgroup/system.slice/docker-abc123.scope/memory.current
# 234881024  ← currently using 224MB

# CPU throttle stats (is the container being throttled?)
cat /sys/fs/cgroup/system.slice/docker-abc123.scope/cpu.stat
# usage_usec 120341234
# throttled_usec 45234123   ← if high, container is CPU-throttled
```

### How Kubernetes Uses Namespaces and cgroups

```yaml
# Pod spec → requests and limits
resources:
  requests:
    memory: "256Mi"   # Scheduler uses this for placement
    cpu: "250m"
  limits:
    memory: "512Mi"   # Sets cgroup memory.max = 536870912 bytes
    cpu: "500m"       # Sets cgroup cpu.max = 50000 1000000
```

```bash
# Kubernetes creates a cgroup hierarchy per pod:
# /sys/fs/cgroup/kubepods/
#   burstable/
#     pod<UID>/
#       <container-ID>/
#         memory.max         ← from limits.memory
#         cpu.max            ← from limits.cpu

# Find a pod's cgroup
CONTAINER_ID=$(kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d/ -f3)
find /sys/fs/cgroup -name "*${CONTAINER_ID:0:12}*" -type d
```

### The OOM Kill → CrashLoopBackOff Connection

```
Container process allocates beyond memory.max
    ↓
cgroup OOM killer fires (not system OOM killer)
    ↓
Container's PID 1 is killed
    ↓
Kubernetes sees: container exit code 137 (128 + SIGKILL)
    ↓
Kubernetes restarts container → CrashLoopBackOff if keeps happening

# How to confirm it was an OOM kill:
kubectl describe pod mypod | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled     ← confirm
#   Exit Code: 137
```

---

---

# Q16 — Container Networking — veth, bridge, CNI

> 🔥 Hard | K8s Networking

## The Question
**"How does container networking work? What is a veth pair? What is a bridge? How does a Kubernetes pod communicate with another pod on a different node?"**

## The Answer

### veth Pairs — Virtual Ethernet Cable

A **veth pair** is a virtual network cable with two ends. Anything written to one end comes out the other. This is how containers get network connectivity:

```
Container namespace:          Host namespace:
┌──────────────────┐         ┌─────────────────────┐
│  eth0 (veth0)    │◄───────►│  veth1               │
│  10.244.0.5/24   │  kernel │  (plugged into       │
└──────────────────┘  memory │   bridge cni0)        │
                             └─────────────────────┘
```

```bash
# Create a veth pair manually (what container runtimes do automatically)
ip link add veth0 type veth peer name veth1

# Move one end into a container's network namespace
ip link set veth0 netns <container_netns>

# Assign IP to the container end
ip netns exec <container_netns> ip addr add 10.244.0.5/24 dev veth0
ip netns exec <container_netns> ip link set veth0 up

# Plug the host end into the bridge
ip link set veth1 master cni0
ip link set veth1 up

# View veth pairs on a Kubernetes node
ip link show type veth
# 15: veth3a4b5c@if14: <BROADCAST,MULTICAST,UP>
# Each running pod has one veth pair
```

### The Bridge (cni0)

All veth host-ends are plugged into a Linux bridge (`cni0` in Flannel/Calico). The bridge acts like a virtual switch — pods on the same node communicate through it.

```bash
# Inspect the bridge
ip link show cni0
bridge link show cni0   # Show all interfaces plugged in
brctl show cni0         # Legacy tool

# A pod on the same node:
# Pod A (10.244.0.5) → veth-a → bridge → veth-b → Pod B (10.244.0.6)
# This is a layer 2 forward — no routing needed
```

### Cross-Node Pod Communication

For Pod A on Node 1 to reach Pod B on Node 2:

**Option 1: Overlay Network (Flannel VXLAN)**
```
Pod A (10.244.0.5) → veth → cni0 bridge → flannel.1 (VXLAN device)
    → VXLAN encapsulates the pod packet in a UDP packet
    → UDP packet sent to Node 2's IP over the host network
    → Node 2 flannel.1 decapsulates → cni0 bridge → veth → Pod B
```

**Option 2: Direct Routing (Calico BGP)**
```
Pod A (10.244.0.5) → veth → routing table → eth0 (Node 1)
    → Node 1 advertises "10.244.0.0/24 is here" via BGP
    → Router forwards packet to Node 2
    → Node 2 routing table → veth → Pod B
```

```bash
# On a Kubernetes node — see pod network routes
ip route
# 10.244.0.0/24 dev cni0  proto kernel  # Same-node pods
# 10.244.1.0/24 via 192.168.1.20 dev eth0  # Pods on node 2
# 10.244.2.0/24 via 192.168.1.21 dev eth0  # Pods on node 3

# Trace a packet's path through iptables (Kubernetes uses iptables heavily)
sudo iptables -t nat -nvL PREROUTING | head -20
# KUBE-SERVICES chain handles Service ClusterIP → Pod IP translation
```

### How Kubernetes Services Work (iptables DNAT)

```bash
# A Service (ClusterIP: 10.96.0.10:80) → Pod (10.244.0.5:8080)
# kube-proxy writes iptables DNAT rules:

sudo iptables -t nat -nvL KUBE-SVC-XXXX
# The DNAT rule: destination 10.96.0.10:80 → 10.244.0.5:8080
# With multiple pods: randomly distributes (DNAT with probability)
```

---

---

# Q17 — Linux Kernel Capabilities in Containers

> 🔥 Hard | K8s Security

## The Question
**"What capabilities does a container have by default? Why is running containers as root dangerous even with namespaces? How do you harden a container's security context in Kubernetes?"**

## The Answer

### Default Container Capabilities (Docker/Kubernetes)

By default, containers drop most capabilities but retain a surprising number:

**Default ALLOWED capabilities:**
```
CAP_AUDIT_WRITE    - write to kernel audit log
CAP_CHOWN          - change file ownership
CAP_DAC_OVERRIDE   - bypass file permission checks  ← dangerous
CAP_FOWNER         - bypass permission checks for file owner
CAP_FSETID         - set SUID/SGID on files
CAP_KILL           - send signals to any process (in same namespace)
CAP_MKNOD          - create special files
CAP_NET_BIND_SERVICE - bind to ports < 1024
CAP_NET_RAW        - raw sockets (tcpdump, ping) ← can be exploited
CAP_SETGID         - change GIDs
CAP_SETUID         - change UIDs
CAP_SETPCAP        - modify process capabilities
CAP_SYS_CHROOT     - chroot()
```

**Why "container root" ≠ "safe":**

Root inside a container with `CAP_DAC_OVERRIDE` + `CAP_SYS_PTRACE` (if not dropped) can read other containers' memory on the same node. A container escape vulnerability (e.g., runc CVE-2019-5736) exploits this: root inside the container overwrites the host's `/proc/self/exe` → code runs on the host as root.

### Hardening in Kubernetes SecurityContext

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true          # Reject if container tries to run as root
    runAsUser: 1000             # Run as UID 1000
    runAsGroup: 3000
    fsGroup: 2000               # Files in mounted volumes owned by this GID
    seccompProfile:
      type: RuntimeDefault      # Enable default seccomp profile (blocks ~300 syscalls)

  containers:
    - name: app
      image: myapp:v1
      securityContext:
        allowPrivilegeEscalation: false  # Cannot gain more privileges (no SUID)
        readOnlyRootFilesystem: true     # Root filesystem is read-only
        capabilities:
          drop:
            - ALL                        # Drop every capability
          add:
            - NET_BIND_SERVICE           # Re-add only what you need
```

### Capability Investigation

```bash
# What capabilities does a running container have?
# Find the container PID
docker inspect <container_id> | jq '.[0].State.Pid'

# Read capability bitmask
cat /proc/<PID>/status | grep Cap
# CapPrm: 00000000a80425fb
# CapEff: 00000000a80425fb

# Decode it
capsh --decode=00000000a80425fb

# From inside the container
cat /proc/self/status | grep Cap
capsh --print    # If capsh is available in the image
```

### Seccomp — Syscall Filtering

Even beyond capabilities, you can restrict which syscalls a container can make:

```bash
# The RuntimeDefault seccomp profile blocks ~300 dangerous syscalls including:
# keyctl, ptrace, clone (with CLONE_NEWUSER), mount, ...

# Check if seccomp is applied to a process
cat /proc/<PID>/status | grep Seccomp
# Seccomp: 2   (0=disabled, 1=strict, 2=filter)
```

---

---

# Q18 — Node Performance Tuning for Kubernetes

> 🔥 Hard | K8s Performance

## The Question
**"What Linux-level tuning do you apply to Kubernetes worker nodes? What are the most impactful sysctl parameters? Why must swap be disabled on K8s nodes?"**

## The Answer

### Why Swap Must Be Disabled on Kubernetes Nodes

The kubelet refuses to start with swap enabled by default (pre-K8s 1.28). The reasons:

1. **Predictable memory limits:** cgroup memory limits assume RAM only. If a pod exceeds its memory limit, the OOM killer fires. With swap, the pod instead swaps to disk — the limit is effectively bypassed
2. **Performance unpredictability:** A pod hitting swap goes from nanosecond RAM access to millisecond disk access — latency spikes 10,000×
3. **Scheduler accuracy:** The scheduler places pods based on memory requests. Swap distorts actual available memory, causing over-scheduling

```bash
# Disable swap immediately
sudo swapoff -a

# Disable permanently
sudo sed -i '/swap/d' /etc/fstab
# or comment out swap line: /swapfile swap swap defaults 0 0

# Verify
free -m | grep Swap
# Swap: 0   0   0  ← all zeros = swap disabled
```

### Critical sysctl Parameters for K8s Nodes

```bash
# /etc/sysctl.d/99-kubernetes.conf

# --- Network tuning ---

# Increase maximum number of tracked connections (for busy nodes)
net.netfilter.nf_conntrack_max = 1048576

# Increase ARP table size (for large clusters — many pod IPs)
net.ipv4.neigh.default.gc_thresh1 = 1024
net.ipv4.neigh.default.gc_thresh2 = 4096
net.ipv4.neigh.default.gc_thresh3 = 8192

# Enable kernel to forward packets between interfaces (required for pods)
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# Increase local port range (more outbound connections from node)
net.ipv4.ip_local_port_range = 1024 65535

# TCP performance
net.core.somaxconn = 65535          # Max connections in accept queue
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65536

# Reduce TIME_WAIT accumulation
net.ipv4.tcp_tw_reuse = 1           # Reuse TIME_WAIT sockets for new connections
net.ipv4.tcp_fin_timeout = 15       # Reduce default 60s FIN wait

# --- Memory ---
vm.swappiness = 0                   # Don't use swap at all
vm.overcommit_memory = 1            # Allow overcommit (needed for certain workloads)
vm.max_map_count = 262144           # Required by Elasticsearch, many JVM apps

# --- File system ---
fs.inotify.max_user_watches = 524288     # Many containers watch files (Prometheus, etc.)
fs.inotify.max_user_instances = 512
fs.file-max = 2097152               # System-wide FD limit

# Apply all settings
sudo sysctl --system
```

### Huge Pages for Databases

Some databases (Oracle, SAP HANA, Elasticsearch with memory locking) use huge pages for better TLB efficiency:

```bash
# Check current huge page config
cat /proc/meminfo | grep Huge
# HugePages_Total: 0
# HugePages_Free:  0
# Hugepagesize:    2048 kB

# Reserve huge pages (2MB each)
echo 512 | sudo tee /proc/sys/vm/nr_hugepages   # Reserve 1GB (512 × 2MB)

# Persist
echo "vm.nr_hugepages = 512" | sudo tee -a /etc/sysctl.d/99-hugepages.conf

# Kubernetes: request huge pages in pod spec
resources:
  limits:
    hugepages-2Mi: 1Gi
  requests:
    hugepages-2Mi: 1Gi
    memory: 1Gi         # Must also request regular memory equal to hugepage amount
```

### Disable Transparent Huge Pages (Required for Most Databases)

```bash
# THP causes latency spikes — disable for database workloads
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Make persistent via rc.local or systemd service
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages
DefaultDependencies=false
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo never > /sys/kernel/mm/transparent_hugepage/enabled"
ExecStart=/bin/sh -c "echo never > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=basic.target
EOF
sudo systemctl enable --now disable-thp
```

### Node Resource Reservations

```yaml
# kubelet config: reserve resources for OS and kubelet itself
# Without this, pods can consume ALL node resources → node goes NotReady

# /var/lib/kubelet/config.yaml
kubeReserved:
  cpu: "200m"
  memory: "200Mi"
  ephemeral-storage: "1Gi"
systemReserved:
  cpu: "500m"
  memory: "500Mi"
  ephemeral-storage: "10Gi"
evictionHard:
  memory.available: "200Mi"        # Evict pods when only 200MB free
  nodefs.available: "10%"
  imagefs.available: "15%"
```

---

---

# Q19 — eBPF — What It Is and Why It Matters for DevOps

> 🔥 Hard | Observability

## The Question
**"What is eBPF? Why is it significant for observability and security in modern Linux/Kubernetes environments? What tools use it?"**

## The Answer

### What eBPF Is

**eBPF (extended Berkeley Packet Filter)** is a technology that lets you run sandboxed programs **inside the Linux kernel** without modifying kernel source code or loading kernel modules.

Before eBPF: to add custom kernel functionality, you had to write a kernel module. Kernel modules can crash the kernel and must be signed. They're dangerous.

With eBPF: you write a small program that the kernel verifies is safe, then runs it in an in-kernel virtual machine. It can't crash the kernel. It can hook into any kernel event: syscalls, network packets, function calls, performance counters.

```
Your eBPF program (C code)
    ↓ (clang compiles to eBPF bytecode)
Kernel verifier (checks for infinite loops, out-of-bounds access)
    ↓
JIT compilation to native instructions
    ↓
Attached to: kprobe, uprobe, tracepoint, XDP, socket filter, cgroup
    ↓
Runs in kernel context — zero copy, nanosecond overhead
```

### Why eBPF Is a Game-Changer

**Without eBPF (traditional monitoring):**
- `strace` uses `ptrace` — stops/restarts the process for every syscall → 2× slowdown
- Network monitoring requires copying packets to userspace
- Adding observability requires modifying application code (adding instrumentation)

**With eBPF:**
- Observe any process without touching its code
- < 1% overhead in most cases
- Kernel-level granularity — see every syscall, every network packet, every function call

### eBPF-Based Tools DevOps Engineers Should Know

**bpf-tools / bcc (BPF Compiler Collection):**
```bash
# Install on Amazon Linux / Ubuntu
sudo yum install bpftrace bcc-tools -y
sudo apt install bpftrace bpfcc-tools -y

# execsnoop: trace all exec() calls system-wide (see every new process)
sudo execsnoop-bpfcc
# PID   PPID  RET ARGS
# 12345 12300   0 /usr/bin/curl https://api.example.com
# ← See exactly what processes are being spawned by your app

# opensnoop: trace all file open() calls
sudo opensnoop-bpfcc -p <PID>

# tcpconnect: trace all TCP connections
sudo tcpconnect-bpfcc
# PID    COMM   IP  SADDR            DADDR            DPORT
# 1234   curl   4   10.0.1.5         10.0.1.10        5432
# ← See every connection your app makes, in real time

# biolatency: disk I/O latency histogram
sudo biolatency-bpfcc 10

# runqlat: CPU scheduler latency (how long processes wait in run queue)
sudo runqlat-bpfcc 10

# profile: CPU flame graphs without stopping processes
sudo profile-bpfcc -adf 30 > out.profile
flamegraph.pl out.profile > flame.svg
```

**bpftrace — one-liner observability:**
```bash
# Count syscalls by process name
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }'

# Trace slow disk reads (>10ms)
sudo bpftrace -e 'kprobe:blk_account_io_start { @start[arg0] = nsecs; }
  kprobe:blk_account_io_done /@start[arg0]/
  { $lat = nsecs - @start[arg0];
    if ($lat > 10000000) { printf("Slow I/O: %d ms\n", $lat/1000000); }
    delete(@start[arg0]); }'

# Who is reading /etc/passwd?
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat
  /str(args->filename) == "/etc/passwd"/
  { printf("PID %d (%s) opened /etc/passwd\n", pid, comm); }'
```

### eBPF in Kubernetes — Cilium and Tetragon

**Cilium** replaces kube-proxy's iptables rules with eBPF programs — dramatically faster for large clusters where iptables rule chains can have 50,000+ entries.

**Tetragon** (by Cilium team) provides runtime security using eBPF:
```yaml
# TracingPolicy: alert on any process writing to /etc/passwd
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-etc-passwd-writes
spec:
  kprobes:
    - call: "security_file_permission"
      syscall: false
      args:
        - index: 0
          type: "file"
        - index: 1
          type: "int"
      selectors:
        - matchArgs:
          - index: 0
            operator: "Equal"
            values:
            - "/etc/passwd"
```

**Pixie** — zero-instrumentation observability using eBPF:
- Automatically captures HTTP/gRPC traffic, DB queries, latency histograms
- No code changes, no sidecars, no sampling
- Ideal for Kubernetes troubleshooting

---

---

# Q20 — Troubleshooting: 5 Classic Production Linux Scenarios

> 🔥 Hard | Debugging

## The Question
**"Walk through how you'd handle these 5 real production scenarios."**

---

### Scenario 1: Server load is 40 but only 8 CPUs — what's happening?

```bash
# Load average = 40, CPUs = 8
# First instinct: all CPUs saturated
# But: load average counts RUNNING + UNINTERRUPTIBLE processes
# Load of 40 with 8 CPUs means 32 processes are stuck in D state!

# Confirm:
ps aux | awk '$8 == "D" {print}' | wc -l
# 35 processes in D state

# What are they waiting on?
cat /proc/$(ps aux | awk '$8=="D"{print $2}' | head -1)/wchan
# nfs_sync_mapping_range   ← NFS mount is stalled!

# Fix:
umount -f -l /mnt/nfs-share    # Force lazy unmount the stuck NFS
# Processes in D state will unblock once the mount is gone
```

---

### Scenario 2: "No space left on device" but df shows 30% free

```bash
# Classic inode exhaustion
df -ih
# Filesystem     Inodes  IUsed IFree IUse% Mounted on
# /dev/xvda1     655360 655360     0  100% /   ← 100% inode usage!

# Find who's using all the inodes
find / -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head
# Result: /var/spool/postfix/maildrop has 600,000 files (mail queue backup)

# Fix: clear the mail queue
sudo postsuper -d ALL deferred
# or just delete all files
sudo find /var/spool/postfix/maildrop -type f -delete
```

---

### Scenario 3: App was running fine, now all connections time out — no code change deployed

```bash
# Check if app is still running
systemctl status myapp   # ← Running, PID 5678

# Can it accept connections?
ss -tlnp | grep :8080   # ← Listening

# Curl from local host
curl -v localhost:8080/health   # ← Works!

# So external connections are timing out but app is fine locally
# → Something is blocking inbound traffic

# Check iptables
sudo iptables -nvL INPUT | head -30
# Zero bytes matching the ACCEPT rule for port 8080...
# Chain INPUT (policy DROP)  ← Firewall was changed!
# Someone set default policy to DROP

# Fix:
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
# Or: find and remove the overly broad DROP rule

# Check security group (if EC2)
aws ec2 describe-security-groups --group-ids sg-xxx
```

---

### Scenario 4: Application is slow, memory and CPU look fine, disk I/O looks fine

```bash
# Network is the last resort — check it
# Is the app waiting on external calls?
strace -p <PID> -e trace=network -T 2>&1 | grep -E "connect|recvfrom|sendto"
# connect(4, {sa_family=AF_INET, sin_addr=inet_addr("10.0.1.10"), sin_port=5432})
# recvfrom(4, ...) = -1 EAGAIN (Resource temporarily unavailable) <2.003241>
#                                                                     ↑ 2 second wait on DB

# Database is slow — check connection latency
time psql -h 10.0.1.10 -U app -c "SELECT 1"
# real 0m3.123s  ← 3 seconds just to connect

# TCP retransmits? (connection quality issues)
netstat -s | grep retransmit
# 45234 segments retransmitted  ← high retransmit count

# MTU mismatch? (common in VPN/overlay networks)
ping -M do -s 1472 10.0.1.10   # Test with max MTU packet
# From 10.0.1.5: frag needed and DF set  ← MTU mismatch!
# Fix: reduce MTU or fix network configuration
```

---

### Scenario 5: A Kubernetes pod keeps restarting with exit code 1, no clear error in logs

```bash
# kubectl logs only shows current container — check previous container logs
kubectl logs mypod --previous

# Describe the pod for full event history
kubectl describe pod mypod

# Check for OOMKill (exit code 137)
kubectl describe pod mypod | grep -A5 "Last State"

# The logs might show nothing if the process fails at startup
# Run the container with a shell to investigate interactively
kubectl debug mypod -it --image=busybox --share-processes -- sh

# Or override the command to keep the pod alive
kubectl run debug-pod \
  --image=myapp:v1 \
  --command -- sleep infinity
kubectl exec -it debug-pod -- bash
# Now run the app manually and watch the error

# Check if a required config/secret is missing
kubectl exec mypod -- env | sort
kubectl exec mypod -- ls /etc/secrets/   # Check mounted secrets exist

# Check for bad init containers
kubectl describe pod mypod | grep -A10 "Init Containers"

# Application can't find a required library
kubectl exec mypod -- ldd /app/binary
# libssl.so.1.1 => not found  ← missing library in container image
```

---

## 📊 Master Summary Table

| Topic | Key Commands | Interview Focus |
|-------|-------------|-----------------|
| Process states | `ps aux`, `top`, `kill -SIGCHLD` | Zombie vs orphan, D state |
| File permissions | `chmod`, `find -perm -4000` | SUID attack surface |
| systemd | `systemctl`, `journalctl`, `systemd-analyze` | Unit file structure, journald |
| Performance | `vmstat`, `iostat`, `mpstat`, `iotop` | USE method, I/O wait vs CPU |
| Memory | `free -m`, `/proc/meminfo`, `dmesg \| grep oom` | OOM killer scoring, swap tradeoffs |
| Networking | `ss`, `iptables`, `tcpdump`, `conntrack` | Connection tracking, debugging dropped packets |
| Boot process | `journalctl -b`, `systemd-analyze blame` | BIOS → GRUB → kernel → systemd |
| File descriptors | `lsof`, `ulimit`, `/proc/<PID>/fd` | "Too many open files" fix |
| LVM | `pvcreate`, `vgextend`, `lvextend -r` | Online resize, snapshot for backup |
| SSH tunneling | `ssh -L/-R/-D`, `ControlMaster` | Local/remote/dynamic forwarding |
| Shell scripting | `set -euo pipefail`, `getopts`, `trap` | Production-safe scripts |
| SELinux/caps | `sestatus`, `audit2why`, `getcap` | Container security hardening |
| Signals | `kill`, `SIGTERM vs SIGKILL` | Graceful shutdown + K8s pod lifecycle |
| strace/lsof | `strace -p -c`, `lsof +L1` | Debugging open files, deleted handles |
| Namespaces/cgroups | `lsns`, `nsenter`, `/sys/fs/cgroup` | How containers work |
| Container networking | `ip link type veth`, `bridge link` | Pod-to-pod communication |
| Capabilities | `getcap`, `capsh --decode`, securityContext | Drop ALL caps in production |
| K8s node tuning | `sysctl`, `swapoff`, kubelet config | Must-have tunables |
| eBPF | `bpftrace`, `execsnoop`, Cilium | Zero-overhead observability |
| Troubleshooting | All of the above | Systematic scenario debugging |

---

*Last Updated: 2025 | Covers Linux 5.x/6.x, Kubernetes 1.28+, Amazon Linux 2023, Ubuntu 22.04+*
