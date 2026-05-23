# Linux Interview Question 2 — Bastion Host & Secure Network Access

> **Topic:** What is a Bastion Host and how does it secure access to private infrastructure
> **Level:** Beginner–Intermediate
> **Relevance:** Asked in almost every DevOps/Cloud/SRE interview — understanding this shows you think about security architecture, not just functionality

---

## ❓ The Question

> **"What is a Bastion host? How does it work, and why would you use one instead of giving users direct access to private servers?"**

A follow-up you'll often get: *"How is it different from a VPN?"* or *"How would you harden a Bastion host?"*

---

## 🧠 The Core Concept

In a typical AWS setup, your EC2 instances live inside a **private VPC subnet** — they have no public IP and cannot be reached directly from the internet. That's intentional. But developers and admins still need SSH access to those machines.

The naive solution is to attach a public IP to every server. That's a terrible idea — it exposes every server to internet-facing attacks.

The **Bastion host** (also called a jump server, jump box, or gateway server) is the **single, hardened entry point** into your private network. Users SSH into the Bastion first, and from there, SSH into the private instance they need. The private servers never touch the internet directly.

```
Internet → Bastion Host (public subnet) → Private EC2s (private subnet)
              ↑
         Only this machine has a public IP
         Only this machine accepts inbound SSH from the internet
```

---

## 🖼️ Reference Diagram (Source: KodeKloud)

![Bastion host managing access to EC2 instances in a VPC — access control diagram](https://kodekloud.com/kk-media/image/upload/v1752873383/notes-assets/images/DevOps-Interview-Preparation-Course-Linux-Question-2/bastion-host-ec2-access-diagram.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **Bastion Host** | A hardened server in a public subnet that acts as the sole SSH entry point to private servers |
| **Jump Server / Jump Box** | Same thing — different names used interchangeably |
| **VPC** | Virtual Private Cloud — isolated network in AWS where your EC2s live |
| **Public Subnet** | Subnet with a route to the internet gateway — Bastion lives here |
| **Private Subnet** | Subnet with no direct internet route — application servers live here |
| **Security Group** | AWS firewall rules at the instance level — controls inbound/outbound traffic |
| **SSH Agent Forwarding** | Allows your local SSH key to authenticate through the Bastion to private servers without copying the key to the Bastion |
| **ProxyJump** | Modern SSH config option that routes SSH through the Bastion transparently |

---

## ⚙️ How It Works — Step by Step

### The Access Flow

```
1. User on laptop  →  SSH to Bastion (public IP, port 22)
                        ↓ (Bastion authenticates user)
2. From Bastion    →  SSH to private EC2 (private IP, port 22)
                        ↓ (private EC2 grants access)
3. User works on private EC2
```

In practice, you don't want users to **manually** SSH into the Bastion and then SSH again. That's clunky. The modern approach uses `ProxyJump` so the whole thing is transparent — one command, one connection, feels like you're SSHing directly to the private instance.

### ProxyJump — Transparent Bastion Access

```bash
# One-liner: SSH to private EC2 through Bastion
ssh -J ec2-user@<BASTION_PUBLIC_IP> ec2-user@<PRIVATE_EC2_IP>

# Or configure it permanently in ~/.ssh/config (much cleaner)
Host bastion
  HostName 54.123.45.67          # Bastion public IP
  User ec2-user
  IdentityFile ~/.ssh/my-key.pem

Host private-app-server
  HostName 10.0.1.50             # Private EC2 IP
  User ec2-user
  IdentityFile ~/.ssh/my-key.pem
  ProxyJump bastion              # Route through bastion automatically
```

Now `ssh private-app-server` connects through the Bastion in one step — the user doesn't even know (or need to know) about the two-hop architecture.

### SSH Agent Forwarding — Never Copy Keys to Bastion

A critical security point: **never store the private key on the Bastion itself**. If the Bastion is compromised, you don't want the attacker to have your keys to all private servers.

Instead, use **SSH agent forwarding** — your local SSH agent handles authentication for both hops, and the key never touches the Bastion's disk.

```bash
# Start SSH agent and add your key locally
eval $(ssh-agent -s)
ssh-add ~/.ssh/my-key.pem

# Connect with agent forwarding enabled
ssh -A -J ec2-user@<BASTION_IP> ec2-user@<PRIVATE_IP>

# Or in ~/.ssh/config
Host bastion
  HostName 54.123.45.67
  User ec2-user
  ForwardAgent yes
```

---

## 🔐 Security Architecture

### AWS Security Group Configuration

This is the exact setup interviewers often want to hear:

```
Bastion Security Group (inbound rules):
  - Port 22 (SSH) from YOUR_OFFICE_IP/32 only   ← NOT 0.0.0.0/0
  - No other inbound rules

Private EC2 Security Group (inbound rules):
  - Port 22 (SSH) from BASTION_SECURITY_GROUP_ID ← reference the SG, not an IP
  - Port 8080 (app) from ALB_SECURITY_GROUP_ID
  - No inbound from 0.0.0.0/0
```

Referencing the Bastion's **security group ID** (not its IP) in the private server rules is cleaner — it works even if the Bastion gets replaced with a new IP.

### The Nuclear Option — Kill Switch

One of the most powerful advantages of a Bastion host: **a single action cuts off ALL external SSH access** to every private server.

```bash
# Emergency: stop the Bastion immediately (cuts all access)
aws ec2 stop-instances --instance-ids i-0abc123def456789

# Or: revoke the security group rule (even faster — no waiting for stop)
aws ec2 revoke-security-group-ingress \
  --group-id sg-0bastion123 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Restore access when ready
aws ec2 authorize-security-group-ingress \
  --group-id sg-0bastion123 \
  --protocol tcp \
  --port 22 \
  --cidr <TRUSTED_IP>/32
```

Without a Bastion, you'd have to revoke access on every single private server individually. With a Bastion, one action protects everything.

### Hardening the Bastion Itself

Since the Bastion is the most exposed machine, it needs to be the most hardened:

```bash
# Disable password authentication — SSH keys only
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Disable root login
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Limit which users can SSH in
echo "AllowUsers ec2-user deploy-bot" | sudo tee -a /etc/ssh/sshd_config

# Enable MFA (Google Authenticator or Duo) for extra auth layer
sudo yum install google-authenticator -y
# Configure per-user: google-authenticator

# Install and configure fail2ban (ban IPs with failed login attempts)
sudo yum install fail2ban -y
sudo systemctl enable fail2ban --now

# Log all commands executed via Bastion
# Add to /etc/profile.d/bastion-audit.sh:
export PROMPT_COMMAND='history -a >(logger -t "bash_history" -p local1.info)'
```

### Centralised Audit Logging

```bash
# Stream Bastion SSH logs to CloudWatch for audit trail
sudo yum install amazon-cloudwatch-agent -y

# /etc/awslogs/awslogs.conf
[/var/log/secure]
file = /var/log/secure
log_group_name = /bastion/ssh-access
log_stream_name = {instance_id}
datetime_format = %b %d %H:%M:%S

# Now every SSH login, logout, and failed attempt is in CloudWatch
# Set up metric filter + alarm for failed login spikes
```

---

## 🌍 Real-World Scenario

**Setup:** A fintech company with 50 EC2s in private subnets across 3 environments (dev, staging, prod). 12 engineers need access. Security audit requirement: every access event must be logged with user identity and timestamp.

**Before Bastion:** Each engineer had individual SSH keys on every server. No central logging. When an engineer left, they had to rotate keys on 50 machines manually.

**After Bastion:**
- Single Bastion per environment (3 total)
- Engineers authenticate to Bastion via their IAM identity (using AWS Systems Manager Session Manager as a modern alternative — no open port 22 at all)
- All sessions logged to CloudWatch automatically
- When engineer leaves: revoke IAM access → they immediately lose access to all environments
- Security group restricts Bastion SSH to the company's VPN IP range only

**Bonus:** The prod Bastion has an `AllowedHours` cron that automatically removes port 22 access outside business hours using a Lambda function — access is physically impossible at 3 AM unless manually re-enabled.

---

## 🔄 Bastion Host vs Alternatives

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Bastion Host** | Public EC2 as SSH gateway | Simple, industry standard, kill-switch | Still has port 22 exposed; needs hardening |
| **AWS SSM Session Manager** | Browser/CLI tunnel via AWS API (no port 22) | No open ports, full audit, IAM-controlled | AWS-specific; requires SSM agent |
| **VPN (OpenVPN/Wireguard)** | Encrypted tunnel puts user on private network | Access to all resources, not just SSH | More complex setup; all traffic through VPN |
| **Zero Trust (Tailscale, Cloudflare Access)** | Identity-aware proxy per resource | Granular, no VPN client needed | Cost; more moving parts |

For most AWS setups today, **AWS Systems Manager Session Manager** is the modern successor to the classic Bastion — it eliminates open ports entirely. But Bastion hosts remain extremely common, and understanding them is foundational before moving to SSM.

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **`0.0.0.0/0` on port 22** | Bastion exposed to entire internet — brute force target | Restrict to office/VPN IP range in security group |
| **Private key on Bastion disk** | Key compromise = all private servers compromised | Use SSH agent forwarding; never copy `.pem` to Bastion |
| **No logging** | Can't audit who accessed what and when | Enable CloudWatch log streaming from `/var/log/secure` |
| **Single Bastion, no HA** | Bastion down = no one can SSH to prod | Multi-AZ Bastions behind an NLB, or use SSM as fallback |
| **Forgetting to update SG after IP change** | Engineers WFH, IP changes, locked out of Bastion | Use Terraform to manage the allowed CIDR; update via PR |
| **Root login enabled** | Attacker can try to brute force root directly | `PermitRootLogin no` in sshd_config — always |

---

## ✅ Best Practices

- Restrict inbound SSH on the Bastion to **specific IP ranges** (office, VPN) — never `0.0.0.0/0`
- Reference the **Bastion's security group ID** (not its IP) in private server rules — survives instance replacement
- Use **SSH agent forwarding** — keys authenticate from your laptop, never touch the Bastion
- Enable **CloudWatch logging** for all SSH sessions on the Bastion — required for SOC 2 / PCI compliance
- Run the Bastion as a **minimal AMI** (Amazon Linux minimal, no desktop, no unnecessary packages)
- Consider **AWS SSM Session Manager** as a modern alternative — zero open ports, IAM-native access control
- Automate Bastion provisioning with Terraform — a destroyed Bastion should be replaceable in minutes

---

## 📋 Quick Reference

```bash
# Connect through Bastion (one command)
ssh -J ec2-user@<BASTION_IP> ec2-user@<PRIVATE_IP>

# ~/.ssh/config for clean setup
Host private-server
  HostName 10.0.1.50
  User ec2-user
  ProxyJump bastion
  ForwardAgent yes

# Kill all access in emergency
aws ec2 stop-instances --instance-ids <BASTION_ID>

# Check who's logged in right now
w
last | head -20
journalctl -u sshd --since "1 hour ago"

# AWS SSM alternative (no Bastion, no port 22)
aws ssm start-session --target i-0abc123def456789
```

---

> **Interview Takeaway:** A Bastion host is a single hardened gateway that gives you centralized access control, centralized logging, and a kill switch for your entire private network. Don't mention it without also mentioning: SSH agent forwarding (no keys on the Bastion), restricting port 22 to specific IPs (not `0.0.0.0/0`), and that AWS SSM Session Manager is the modern evolution — it removes the open port entirely. That combination shows you understand both the pattern and where the industry is heading.
