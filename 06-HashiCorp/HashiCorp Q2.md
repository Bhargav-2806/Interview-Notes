# HashiCorp Interview Question 2 — Golden Images with HashiCorp Packer

> **Topic:** Using HashiCorp Packer to build consistent, security-hardened machine images (Golden Images) for EC2 and multi-cloud environments
> **Level:** Intermediate
> **Relevance:** Common in interviews for defense, fintech, healthcare, and any regulated environment — shows you understand immutable infrastructure and security hardening at the image level

---

## ❓ The Question

> **"Your security team requires specific security packages to be installed on every EC2 instance. How do you guarantee these packages are always present, consistently, across all environments?"**

Follow-ups you'll commonly hear:
- *"What is a golden image? How do you build and maintain one?"*
- *"How is HashiCorp Packer different from just baking a custom AMI manually?"*
- *"How do you test a Packer-built image before deploying it?"*
- *"How does this fit into a CI/CD pipeline?"*
- *"What's the difference between mutable and immutable infrastructure?"*

---

## 🧠 The Core Problem — Why You Can't Rely on User Data Scripts

The tempting solution is to install security packages via EC2 User Data at launch time:

```bash
#!/bin/bash
# User data script — runs at instance launch
yum install -y aide clamav auditd fail2ban
systemctl enable auditd --now
```

**Why this fails for compliance:**
- Runs at launch → if the package repo is unreachable, packages don't install → instance launches non-compliant with no one noticing
- Every instance wastes 2–5 minutes installing packages on startup
- No way to verify which instances have which versions installed
- In a security audit: "prove every instance has CIS-hardened packages" — you can't, because installation state is ephemeral
- Internet access from launch subnet may be restricted for security reasons (common in defense/regulated environments)

**The right answer:** Bake packages into the AMI before deployment. Every instance launched from it is already compliant on boot — no internet access needed, no installation time, no compliance drift.

---

## 🖼️ Reference Diagram (Source: KodeKloud)

![HashiCorp Packer — JSON/HCL template to OS image, multi-platform golden image creation](https://kodekloud.com/kk-media/image/upload/v1752873357/notes-assets/images/DevOps-Interview-Preparation-Course-Hashicorp-Question-2/hashicorp-packer-machine-images-diagram.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **Packer** | HashiCorp tool that builds machine images from a declarative template |
| **Golden Image** | A pre-baked, security-hardened, version-controlled machine image used as the base for all instances |
| **AMI** | Amazon Machine Image — EC2's format for machine images |
| **Builder** | Packer component that creates the base VM (AWS, Azure, GCP, VMware, Docker) |
| **Provisioner** | Packer component that configures the VM (shell scripts, Ansible, Chef) |
| **Post-processor** | Packer component that handles the artifact after build (copy AMI to regions, create manifest) |
| **HCL2** | HashiCorp Configuration Language v2 — modern Packer template format (replaced JSON) |
| **Immutable Infrastructure** | Servers are never modified after deployment — replaced with new instances from updated images |
| **CIS Benchmark** | Center for Internet Security hardening guidelines — the most common security standard for images |
| **AMI Lifecycle** | Process of building, testing, deprecating, and deleting AMI versions over time |

---

## ⚙️ What Packer Does — The Build Flow

```
Packer Template (HCL2)
         ↓
1. Builder: Launch a temporary EC2 instance from base AMI
         ↓
2. Provisioner: SSH in and run configuration (install packages, harden OS)
         ↓
3. Packer: Create AMI snapshot from the configured instance
         ↓
4. Packer: Terminate the temporary EC2 instance
         ↓
5. Post-processor: Copy AMI to other regions, tag it, write manifest
         ↓
Golden AMI ready → used as base for all future EC2 instances
```

Every subsequent instance launched from this AMI already has everything baked in — no installation, no configuration drift, no compliance gaps.

---

## 🛠️ Modern Packer Template — HCL2 (Not JSON)

> **Note:** The KodeKloud diagram shows JSON format (the old way). Modern Packer uses HCL2, which is what you should demonstrate in interviews.

### Full Production Packer Template

```hcl
# golden-image.pkr.hcl

# ── Variables ─────────────────────────────────────────────────────────────
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "base_ami" {
  type        = string
  description = "Base Amazon Linux 2023 AMI — update quarterly"
  default     = "ami-0c02fb55956c7d316"   # Amazon Linux 2023
}

variable "image_version" {
  type    = string
  default = "1.0.0"
}

variable "environment" {
  type    = string
  default = "production"
}

# ── Local values ──────────────────────────────────────────────────────────
locals {
  timestamp  = regex_replace(timestamp(), "[- TZ:]", "")
  image_name = "golden-image-${var.image_version}-${local.timestamp}"
}

# ── Source: AWS Builder ───────────────────────────────────────────────────
source "amazon-ebs" "golden_image" {
  region        = var.aws_region
  source_ami    = var.base_ami
  instance_type = "t3.medium"    # Temporary build instance (terminated after build)

  # Use SSM instead of SSH (no open port 22 needed)
  communicator                 = "ssh"
  ssh_username                 = "ec2-user"
  ssh_interface                = "session_manager"   # No inbound SSH port required
  iam_instance_profile         = "packer-build-role"  # IAM role for SSM access

  # VPC configuration (build in private subnet)
  vpc_id            = "vpc-0abc123"
  subnet_id         = "subnet-0def456"
  security_group_id = "sg-0ghi789"

  # The final AMI
  ami_name        = local.image_name
  ami_description = "Security-hardened golden image v${var.image_version}"

  # Encrypt the AMI with KMS (required for compliance)
  encrypt_boot = true
  kms_key_id   = "arn:aws:kms:us-east-1:123456789:key/abc-def"

  # Shared with other AWS accounts (e.g., dev/staging/prod accounts)
  ami_users = ["111111111111", "222222222222"]

  # Tags for the AMI and snapshot
  tags = {
    Name            = local.image_name
    Version         = var.image_version
    Environment     = var.environment
    BuildDate       = local.timestamp
    BaseAMI         = var.base_ami
    CISCompliant    = "true"
    ManagedBy       = "packer"
  }

  # Tag the temporary build instance too
  run_tags = {
    Name      = "packer-builder-${local.timestamp}"
    ManagedBy = "packer"
    Temporary = "true"
  }
}

# ── Build block ───────────────────────────────────────────────────────────
build {
  name    = "golden-image-build"
  sources = ["source.amazon-ebs.golden_image"]

  # ── Step 1: Update the OS ──
  provisioner "shell" {
    inline = [
      "echo 'Starting OS update...'",
      "sudo dnf update -y",
      "sudo dnf upgrade -y",
    ]
    # Retry in case of transient package repo issues
    max_retries = 3
  }

  # ── Step 2: Install required security packages ──
  provisioner "shell" {
    inline = [
      # Intrusion detection
      "sudo dnf install -y aide",
      "sudo aide --init",
      "sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz",

      # Antivirus
      "sudo dnf install -y clamav clamav-update",
      "sudo freshclam",

      # Audit daemon (required by NIST/CIS)
      "sudo dnf install -y audit auditd",
      "sudo systemctl enable auditd",

      # Fail2ban
      "sudo dnf install -y fail2ban",
      "sudo systemctl enable fail2ban",

      # CloudWatch agent
      "sudo dnf install -y amazon-cloudwatch-agent",

      # SSM agent (for Systems Manager access — avoids need for bastion)
      "sudo dnf install -y amazon-ssm-agent",
      "sudo systemctl enable amazon-ssm-agent",

      # Compliance and hardening utilities
      "sudo dnf install -y openscap-scanner scap-security-guide",
    ]
  }

  # ── Step 3: Upload and apply CIS hardening scripts ──
  provisioner "file" {
    source      = "./scripts/cis-hardening/"
    destination = "/tmp/cis-hardening/"
  }

  provisioner "shell" {
    script = "./scripts/apply-cis-benchmark.sh"
    environment_vars = [
      "CIS_LEVEL=2",
      "DISABLE_USB=true",
    ]
  }

  # ── Step 4: Apply configuration with Ansible (optional but common) ──
  provisioner "ansible" {
    playbook_file = "./ansible/security-baseline.yml"
    extra_arguments = [
      "--extra-vars", "env=${var.environment}",
      "--tags", "security,hardening,monitoring",
    ]
  }

  # ── Step 5: Run OpenSCAP compliance scan — FAIL BUILD if not compliant ──
  provisioner "shell" {
    inline = [
      "sudo oscap xccdf eval \\",
      "  --profile xccdf_org.ssgproject.content_profile_cis_server_l2 \\",
      "  --report /tmp/compliance-report.html \\",
      "  --results /tmp/compliance-results.xml \\",
      "  /usr/share/xml/scap/ssg/content/ssg-amzn2023-ds.xml || true",
      # Parse results and fail build if score < 90%
      "python3 /tmp/cis-hardening/check-compliance-score.py /tmp/compliance-results.xml 90",
    ]
  }

  # ── Step 6: Collect compliance report as artifact ──
  provisioner "file" {
    source      = "/tmp/compliance-report.html"
    destination = "./artifacts/compliance-report-${local.timestamp}.html"
    direction   = "download"
  }

  # ── Step 7: Final cleanup ──
  provisioner "shell" {
    inline = [
      # Remove build tools and temp files
      "sudo dnf remove -y gcc make",
      "sudo rm -rf /tmp/* /var/tmp/*",
      # Clear shell history
      "history -c",
      "sudo bash -c 'cat /dev/null > /root/.bash_history'",
      # Remove SSH authorized keys (will be re-injected by EC2 key pair)
      "sudo rm -f /home/ec2-user/.ssh/authorized_keys",
      # Remove cloud-init artifacts (forces re-run on first boot)
      "sudo rm -rf /var/lib/cloud/instances/*",
      # Sync and prepare for snapshotting
      "sync",
      "echo 'Image preparation complete'",
    ]
  }

  # ── Post-processors ──────────────────────────────────────────────────────

  # Write a manifest file (useful for CI: captures the AMI ID)
  post-processor "manifest" {
    output     = "manifest.json"
    strip_path = true
  }

  # Copy AMI to other regions (for multi-region deployments)
  post-processor "amazon-ami-management" {
    regions            = ["us-west-2", "eu-west-1"]
    keep_releases      = 3   # Keep last 3 AMI versions in each region
  }
}
```

---

## 📋 CIS Hardening Script Example

```bash
#!/bin/bash
# apply-cis-benchmark.sh — CIS Level 2 hardening (key items)
set -euo pipefail

echo "Applying CIS Benchmark Level ${CIS_LEVEL:-1} hardening..."

# ── Filesystem hardening ──
# Disable unused filesystems
for fs in cramfs freevxfs jffs2 hfs hfsplus squashfs udf; do
    echo "install $fs /bin/true" | sudo tee -a /etc/modprobe.d/disable-filesystems.conf
done

# Set noexec on /tmp
echo "tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev 0 0" | sudo tee -a /etc/fstab

# ── Kernel parameters ──
cat << 'EOF' | sudo tee /etc/sysctl.d/99-cis-hardening.conf
# Prevent IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable IP source routing
net.ipv4.conf.all.accept_source_route = 0

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Enable SYN flood protection
net.ipv4.tcp_syncookies = 1

# Disable IPv6 if not needed
net.ipv6.conf.all.disable_ipv6 = 1

# Protect against SUID core dumps
fs.suid_dumpable = 0

# Address space layout randomization
kernel.randomize_va_space = 2
EOF
sudo sysctl --system

# ── SSH hardening ──
sudo tee /etc/ssh/sshd_config.d/99-cis.conf << 'EOF'
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
MaxAuthTries 4
ClientAliveInterval 300
ClientAliveCountMax 0
LoginGraceTime 60
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
Banner /etc/issue.net
EOF

# ── Audit rules (CIS requirement) ──
sudo tee /etc/audit/rules.d/99-cis.rules << 'EOF'
# Audit privileged commands
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged

# Audit file deletion
-a always,exit -F arch=b64 -S unlinkat -F auid>=1000 -F auid!=4294967295 -k delete

# Audit sudoers changes
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# Audit login/logout
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins
EOF
sudo service auditd restart

echo "CIS hardening complete."
```

---

## 🔄 Packer in a CI/CD Pipeline

The golden image should be rebuilt automatically — on a schedule (quarterly OS updates) or when the base AMI changes (security patches):

```yaml
# .github/workflows/build-golden-image.yml
name: Build Golden Image

on:
  schedule:
    - cron: '0 2 1 * *'    # Monthly on the 1st at 2 AM
  push:
    branches: [main]
    paths:
      - 'packer/**'
      - 'scripts/**'
      - 'ansible/**'
  workflow_dispatch:        # Manual trigger
    inputs:
      image_version:
        description: 'Image version (e.g., 1.2.0)'
        required: true

jobs:
  build-ami:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # OIDC for AWS auth
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — no long-lived keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/packer-build-role
          aws-region: us-east-1

      - name: Setup Packer
        uses: hashicorp/setup-packer@main
        with:
          version: "1.10.0"

      - name: Packer Init
        run: packer init packer/golden-image.pkr.hcl

      - name: Packer Validate
        run: |
          packer validate \
            -var "image_version=${{ github.event.inputs.image_version || '1.0.0' }}" \
            packer/golden-image.pkr.hcl

      - name: Packer Build
        id: packer_build
        run: |
          packer build \
            -var "image_version=${{ github.event.inputs.image_version || '1.0.0' }}" \
            packer/golden-image.pkr.hcl

      - name: Extract AMI ID from manifest
        id: get_ami
        run: |
          AMI_ID=$(jq -r '.builds[-1].artifact_id' manifest.json | cut -d: -f2)
          echo "ami_id=$AMI_ID" >> $GITHUB_OUTPUT
          echo "Built AMI: $AMI_ID"

      - name: Run AMI Tests (Goss / InSpec)
        run: |
          # Launch a test instance from the new AMI
          INSTANCE_ID=$(aws ec2 run-instances \
            --image-id ${{ steps.get_ami.outputs.ami_id }} \
            --instance-type t3.micro \
            --iam-instance-profile Name=test-role \
            --query 'Instances[0].InstanceId' \
            --output text)

          # Wait for it to be ready
          aws ec2 wait instance-status-ok --instance-ids $INSTANCE_ID

          # Run compliance tests via SSM
          aws ssm send-command \
            --instance-ids $INSTANCE_ID \
            --document-name "AWS-RunShellScript" \
            --parameters commands=["goss validate --format tap"] \
            --output text

          # Terminate test instance
          aws ec2 terminate-instances --instance-ids $INSTANCE_ID

      - name: Update Launch Template with new AMI
        run: |
          aws ec2 create-launch-template-version \
            --launch-template-name my-app-template \
            --source-version '$Latest' \
            --launch-template-data "{\"ImageId\":\"${{ steps.get_ami.outputs.ami_id }}\"}"

      - name: Upload compliance report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: compliance-report
          path: artifacts/compliance-report-*.html
```

---

## 🧪 Testing Golden Images — Goss

[Goss](https://github.com/goss-org/goss) is a lightweight YAML-based server testing tool. Run it as a Packer provisioner to validate the image before it's published:

```yaml
# goss.yaml — defines what MUST be true in the golden image
package:
  aide:
    installed: true
    versions:
    - "0.16"
  clamav:
    installed: true
  auditd:
    installed: true
  amazon-cloudwatch-agent:
    installed: true
  amazon-ssm-agent:
    installed: true

service:
  auditd:
    enabled: true
    running: true
  amazon-ssm-agent:
    enabled: true
    running: true
  fail2ban:
    enabled: true

file:
  /etc/ssh/sshd_config.d/99-cis.conf:
    exists: true
    contains:
      - "PermitRootLogin no"
      - "PasswordAuthentication no"
      - "Protocol 2"

  /etc/sysctl.d/99-cis-hardening.conf:
    exists: true
    contains:
      - "net.ipv4.tcp_syncookies = 1"
      - "kernel.randomize_va_space = 2"

kernel-param:
  net.ipv4.tcp_syncookies:
    value: "1"
  kernel.randomize_va_space:
    value: "2"

user:
  ec2-user:
    exists: true
    uid: 1000
    shell: /bin/bash
```

```hcl
# In the Packer template — run Goss after provisioning
provisioner "goss" {
  vars_file   = "./goss-vars.yaml"
  tests       = ["./goss.yaml"]
  format      = "tap"
  # Build fails if any test fails — image is NOT published
}
```

---

## 🔄 Packer vs Custom AMI (Manual) vs User Data

| Approach | Consistency | Auditability | Launch Speed | Compliance Proof | Multi-Cloud |
|----------|------------|-------------|--------------|-----------------|-------------|
| **Manual custom AMI** | Low (someone SSH'd in) | None | Fast | Hard | No |
| **User Data scripts** | Medium (depends on network) | Minimal | Slow (installs on boot) | Very hard | No |
| **HashiCorp Packer** | High (reproducible build) | Full (Git history + CI logs) | Fast (pre-baked) | Easy (scan results in CI) | Yes |
| **Packer + Ansible** | Very high | Complete | Fast | Complete | Yes |

---

## 🌍 Real-World Scenario

**Project:** A defense contractor required all EC2 instances to meet NIST 800-171 compliance with documented proof per audit cycle.

**Problem:** 12 engineering teams each created their own EC2 instances with slightly different configurations. Security scans routinely flagged 3–5 non-compliant configurations across the fleet.

**Solution with Packer:**
1. Security team defined a Packer template with CIS Level 2 hardening, required packages (AIDE, auditd, ClamAV), and an OpenSCAP scan as the final build step
2. Pipeline runs monthly and on every push to the `golden-image` repo
3. Build fails if OpenSCAP compliance score < 90% — image is not published
4. All 12 teams are required to use the latest golden AMI as their base (enforced via SCP: no EC2 launch allowed with AMIs not tagged `CISCompliant=true`)
5. Quarterly audit: auditor gets the CI pipeline link + compliance HTML report + AMI ID — all evidence in one place

**Result:** Compliance findings dropped from 15–20 per quarter to zero. Audit preparation time from 3 days to 2 hours.

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **Build instance left running** | Packer fails mid-build → temporary EC2 instance not terminated | Set up a Lambda or tag-based cleanup; use `packer build -on-error=abort` |
| **Using JSON templates** | JSON is the old format — no variables interpolation, no reuse | Migrate to HCL2: `packer hcl2_upgrade template.json` |
| **Not cleaning cloud-init** | New instances launched from AMI skip first-boot init | Always run `sudo rm -rf /var/lib/cloud/instances/*` at end of provisioning |
| **AMI proliferation** | Hundreds of old AMIs accumulate in the account — storage costs, confusion | Use `amazon-ami-management` post-processor with `keep_releases = 3` |
| **Compliance scan passes, runtime fails** | CIS hardening breaks the application (e.g., /tmp noexec breaks Java) | Test the app on a Packer-built instance in staging before enforcing in prod |
| **Build in public subnet** | Build instance in public subnet = unnecessary internet exposure | Build in private subnet with NAT gateway; use SSM for connectivity instead of SSH |

---

## ✅ Best Practices

- **Use HCL2 templates**, not JSON — parameterized, reusable, version-controlled like application code
- **Build in a private subnet** using SSM Session Manager as the communicator — no port 22 exposed to internet
- **Encrypt the AMI and snapshots** with KMS — required for most compliance frameworks
- **Run a compliance scan inside Packer** (OpenSCAP/InSpec/Goss) — fail the build if standards aren't met, so a non-compliant image can never be published
- **Tag AMIs with metadata**: version, build date, base AMI ID, CIS compliance level — enables governance via SCP
- **Rotate golden images** on a schedule (monthly or quarterly) to pick up OS security patches
- **Keep the last N AMIs** via lifecycle policy — enough to roll back if a new image has issues
- **Store Packer templates in Git** — treat them as code, require PRs and code review for changes

---

## 📋 Quick Reference

```bash
# Install Packer
brew install packer          # Mac
sudo dnf install -y packer   # Amazon Linux

# Initialize (download providers/plugins)
packer init golden-image.pkr.hcl

# Validate template syntax
packer validate golden-image.pkr.hcl

# Dry-run with variable overrides
packer validate \
  -var "image_version=1.2.0" \
  -var "aws_region=us-east-1" \
  golden-image.pkr.hcl

# Build the image
packer build golden-image.pkr.hcl

# Build with debug output
packer build -debug golden-image.pkr.hcl

# Build only a specific source (if template has multiple)
packer build -only="amazon-ebs.golden_image" golden-image.pkr.hcl

# Upgrade an old JSON template to HCL2
packer hcl2_upgrade old-template.json

# Read AMI ID from manifest after build
jq -r '.builds[-1].artifact_id' manifest.json
```

---

> **Interview Takeaway:** The answer isn't "use User Data scripts" — those are fragile, slow, and unauditable. The answer is a **Packer-built golden image**: packages baked in before deployment, compliance scan run as part of the build (fails if below threshold), all tracked in Git and CI. Then show you've thought about the full lifecycle — encrypted AMIs, private subnet builds with SSM, AMI rotation schedule, and governance via SCP that blocks non-compliant AMI launches. That's the complete, enterprise-grade answer.
