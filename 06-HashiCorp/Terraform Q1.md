# Terraform Interview Question 1 — Remote Execution & Provisioners

> **Topic:** Terraform `remote-exec` and `local-exec` provisioners — what they do, when to use them, and critically, when NOT to use them
> **Level:** Intermediate
> **Relevance:** Interviewers often ask about provisioners to test whether you know the right tool for the job — a strong answer explains both how they work AND why you should prefer alternatives in most production scenarios

---

## ❓ The Question

> **"What is the `remote-exec` provisioner in Terraform? When would you use it? How does it differ from `local-exec`?"**

A strong follow-up you'll often get:
- *"HashiCorp actually recommends against using provisioners — why? What should you use instead?"*
- *"How does `remote-exec` connect to an EC2 instance? What does the `connection` block do?"*
- *"What is a `null_resource` and how do you use it with provisioners?"*

---

## 🧠 What Remote-Exec Does

Terraform's job is to provision infrastructure — create an EC2 instance, set up a VPC, attach an EBS volume. But what if you need to **configure** that infrastructure immediately after it's created? That's where provisioners come in.

`remote-exec` SSH (or WinRM) into a freshly created resource and runs commands on it — all as part of the same `terraform apply`. No separate step, no manual intervention.

```
terraform apply
    ↓
1. EC2 instance created (aws_instance resource)
    ↓
2. Terraform opens SSH connection to the instance
    ↓
3. remote-exec runs commands on the instance
    ↓
4. Terraform marks resource as created
```

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **Provisioner** | Terraform block that runs actions on a local or remote machine after a resource is created |
| **`remote-exec`** | Provisioner that SSH/WinRM into a remote resource and runs commands |
| **`local-exec`** | Provisioner that runs commands on the machine running Terraform (your laptop or CI runner) |
| **`connection` block** | Defines how Terraform connects to the remote resource (SSH key, username, host) |
| **`null_resource`** | A resource with no real infrastructure — used purely to attach provisioners or trigger re-runs |
| **`triggers`** | Map of values on a `null_resource`; resource is re-created (and provisioners re-run) when any value changes |
| **`file` provisioner** | Uploads a file or directory from local machine to the remote resource |
| **User Data** | EC2-native mechanism to run scripts at first boot — the preferred alternative to remote-exec |
| **`create` vs `destroy` provisioner** | By default provisioners run on create; `when = destroy` makes them run before resource deletion |

---

## ⚙️ How remote-exec Works — Three Forms

### Form 1: `inline` — List of Commands

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deployer.key_name

  # Connection block — tells Terraform how to SSH in
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/my-key.pem")
    host        = self.public_ip     # self refers to this resource
    timeout     = "5m"
  }

  provisioner "remote-exec" {
    inline = [
      "sudo dnf update -y",
      "sudo dnf install -y nginx",
      "sudo systemctl enable nginx --now",
      "echo 'Nginx installed and running'",
    ]
  }
}
```

### Form 2: `script` — Upload and Run a Single Script

```hcl
provisioner "remote-exec" {
  # Terraform uploads this script to the remote machine, then executes it
  script = "${path.module}/scripts/bootstrap.sh"
}
```

### Form 3: `scripts` — Multiple Scripts in Order

```hcl
provisioner "remote-exec" {
  scripts = [
    "${path.module}/scripts/01-update-os.sh",
    "${path.module}/scripts/02-install-agent.sh",
    "${path.module}/scripts/03-configure-app.sh",
  ]
}
```

### The `connection` Block — SSH and WinRM

```hcl
# SSH connection (Linux)
connection {
  type        = "ssh"
  user        = "ec2-user"
  private_key = file("~/.ssh/key.pem")
  host        = self.public_ip
  port        = 22
  timeout     = "10m"    # How long to wait for SSH to become available
  agent       = false    # Don't use local SSH agent
}

# Bastion host connection (SSH through a jump host)
connection {
  type         = "ssh"
  user         = "ec2-user"
  private_key  = file("~/.ssh/key.pem")
  host         = self.private_ip          # Connect to private IP...
  bastion_host = aws_instance.bastion.public_ip  # ...via bastion
  bastion_user = "ec2-user"
  bastion_private_key = file("~/.ssh/key.pem")
}

# WinRM connection (Windows)
connection {
  type     = "winrm"
  user     = "Administrator"
  password = var.admin_password
  host     = self.public_ip
  https    = true
  insecure = false
  timeout  = "10m"
}
```

---

## 🖥️ `local-exec` — Run Commands Locally

`local-exec` runs on the machine executing Terraform (your workstation or CI runner), not on the provisioned resource. Useful for triggering external processes after infrastructure is created.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"

  # After the EC2 instance is created, register it in an external CMDB
  provisioner "local-exec" {
    command = "python3 scripts/register-cmdb.py ${self.id} ${self.private_ip}"
  }

  # Run Ansible against the new instance
  provisioner "local-exec" {
    command = "ansible-playbook -i '${self.public_ip},' playbooks/bootstrap.yml"
    environment = {
      ANSIBLE_HOST_KEY_CHECKING = "false"
    }
  }
}
```

```hcl
# local-exec on destroy — trigger cleanup in external system
resource "aws_instance" "web" {
  # ...

  provisioner "local-exec" {
    when    = destroy   # Runs BEFORE the resource is destroyed
    command = "python3 scripts/deregister-cmdb.py ${self.id}"
  }
}
```

### Using `local-exec` with `working_dir` and Interpreter

```hcl
provisioner "local-exec" {
  command     = "invoke deploy --env production"
  working_dir = "${path.module}/deployment/"
  interpreter = ["python3", "-c"]   # Override default shell
  # interpreter = ["/bin/bash", "-c"]  # Explicit bash
}
```

---

## 🏗️ `null_resource` — Provisioners Without a Real Resource

Sometimes you need to run a provisioner that isn't tied to a specific resource — or you want to re-trigger a provisioner when a variable changes (not just when the resource is re-created).

```hcl
resource "null_resource" "configure_db" {
  # Re-run the provisioner whenever these values change
  triggers = {
    db_endpoint = aws_db_instance.main.endpoint
    config_hash = filemd5("${path.module}/scripts/configure-db.sql")
  }

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/key.pem")
    host        = aws_instance.bastion.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "mysql -h ${aws_db_instance.main.endpoint} -u admin < /tmp/configure-db.sql",
    ]
  }

  # Ensure DB and EC2 are created before this runs
  depends_on = [
    aws_db_instance.main,
    aws_instance.bastion,
  ]
}
```

The `triggers` map is key — if `config_hash` changes (you updated the SQL file), Terraform destroys and re-creates the `null_resource`, running the provisioner again. This is how you make provisioners re-run on config changes.

---

## ⚠️ The Critical Point — HashiCorp Recommends Against Provisioners

This is the most important thing to say in an interview after explaining provisioners:

> **HashiCorp's own documentation states: "Provisioners are a last resort."**

### Why Provisioners Are Problematic

**1. They break the Terraform state model**

Terraform tracks infrastructure state — what exists and what should exist. Provisioner execution is not tracked. If `remote-exec` fails halfway through, Terraform marks the resource as tainted (must be destroyed and re-created) but has no idea what state the machine is in.

**2. They require network connectivity at apply time**

`remote-exec` needs SSH access to the instance. In a typical production setup with security groups that block SSH from the CI runner, or instances in private subnets, this fails.

**3. They create hidden dependencies**

The configuration applied by `remote-exec` is invisible to Terraform's dependency graph and state. If you change the script and run `terraform apply`, Terraform has no way to know the instance needs updating — it's already "created."

**4. They couple provisioning and configuration**

Terraform should provision infrastructure. Configuration management (installing software, managing files) belongs in dedicated tools.

### What to Use Instead

| Provisioner Use Case | Better Alternative |
|----------------------|--------------------|
| Install packages on first boot | `user_data` (EC2) / `cloud-init` |
| Install security packages consistently | HashiCorp Packer (baked into AMI) |
| Complex server configuration | Ansible (run separately or via CI) |
| Register resource in external system | Terraform `terraform_data` / event-based Lambda |
| Run SQL migrations after DB creation | `local-exec` calling a migration script (acceptable) |
| Install an agent on every instance | Bake it into the golden AMI with Packer |

---

## ✅ The Preferred Approach — `user_data` over `remote-exec`

For bootstrapping EC2 instances, `user_data` is almost always better:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"

  # No SSH needed — cloud-init runs this at first boot as root
  user_data = <<-EOF
    #!/bin/bash
    set -euo pipefail
    dnf update -y
    dnf install -y nginx amazon-cloudwatch-agent
    systemctl enable nginx --now
    /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
      -a fetch-config -m ec2 -c ssm:/cloudwatch-agent-config -s
  EOF

  # Or reference an external file
  # user_data = file("${path.module}/scripts/bootstrap.sh")

  # Or use templatefile for variable substitution
  # user_data = templatefile("${path.module}/scripts/bootstrap.sh.tpl", {
  #   environment = var.environment
  #   app_version = var.app_version
  # })
}
```

**Why `user_data` is better than `remote-exec`:**
- Runs at instance boot — no SSH connection from Terraform required
- Works in private subnets with no inbound connectivity
- Idempotent by design — the instance starts configured
- Logs go to `/var/log/cloud-init-output.log` for debugging
- Works with Auto Scaling Groups (each instance bootstraps itself)

---

## 🌍 Real-World Scenario: When remote-exec IS Acceptable

Despite the caveats, `remote-exec` has legitimate use cases when the alternatives don't fit.

**Scenario:** A team uses Terraform to provision on-premises VMs via VMware vSphere. These VMs don't have a cloud-init equivalent. After Terraform creates the VM, `remote-exec` installs the monitoring agent and registers with the CMDB — both steps that must happen before the VM is considered ready.

```hcl
resource "vsphere_virtual_machine" "app_server" {
  # ... vSphere config ...

  connection {
    type        = "ssh"
    user        = "svc-terraform"
    private_key = file(var.ssh_private_key_path)
    host        = self.guest_ip_addresses[0]
    timeout     = "10m"
  }

  provisioner "file" {
    source      = "${path.module}/scripts/install-agent.sh"
    destination = "/tmp/install-agent.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/install-agent.sh",
      "sudo /tmp/install-agent.sh ${var.monitoring_server}",
      "sudo /opt/agent/bin/register-cmdb.sh ${var.cmdb_endpoint}",
      "rm /tmp/install-agent.sh",
    ]
  }
}
```

This is reasonable because:
- There's no `user_data` equivalent on VMware
- The steps are required for compliance before the VM is considered provisioned
- The SSH connection is internal (data center network, not internet)

---

## 🔄 Provisioner Behavior — Creation vs Destroy, `on_failure`

```hcl
provisioner "remote-exec" {
  # Default: when = "create" — runs after resource is created

  inline = ["sudo systemctl restart myapp"]

  # on_failure behavior:
  on_failure = continue   # Don't fail the apply if this step fails
  # on_failure = fail     # Default: mark resource as tainted and fail apply
}

provisioner "local-exec" {
  when    = destroy   # Runs BEFORE the resource is destroyed
  command = "deregister-service.sh ${self.id}"

  on_failure = continue   # Don't block destroy if deregistration fails
}
```

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **SSH not ready yet** | Instance created but SSH daemon not up → connection timeout | Add `timeout = "5m"` in connection block; use `user_data` for early-boot setup |
| **Private IP not reachable** | CI runner can't SSH to private subnet instance | Use bastion connection, or switch to `user_data` |
| **`self.public_ip` is null** | Instance has no public IP (private subnet) | Use `self.private_ip` with bastion, or use SSM Session Manager |
| **Provisioner re-runs on every apply** | Changed an attribute that forces re-creation | Provisioners run on create — use `null_resource` + `triggers` for re-run control |
| **State shows success but config wasn't applied** | Script failed silently (no `set -e`) | Always add `set -euo pipefail` to scripts; use `on_failure = fail` |
| **Hard to debug failures** | Provisioner output hard to capture | Add logging to scripts: `exec > >(tee -a /var/log/terraform-provision.log) 2>&1` |

---

## ✅ Best Practices

- **Prefer `user_data` over `remote-exec`** for EC2 instance bootstrapping — no SSH needed, works in private subnets, works with ASGs
- **Use Packer** for anything that should be consistent across all instances — bake it into the AMI
- **Use `remote-exec` as a last resort** — on-premises VMs, non-cloud platforms, or when `user_data` genuinely isn't available
- **Always use `set -euo pipefail`** in scripts passed to `remote-exec` — silent failures in provisioners are hard to detect
- **Add `timeout`** to connection blocks — default timeout can leave Terraform hanging for minutes before failing
- **Use `null_resource` + `triggers`** when you need to re-run configuration logic after the initial creation
- **`on_failure = continue`** for destroy provisioners — never let a failed deregistration block resource cleanup

---

## 📋 Quick Reference

```hcl
# remote-exec with inline commands
provisioner "remote-exec" {
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/key.pem")
    host        = self.public_ip
    timeout     = "5m"
  }
  inline = ["sudo dnf install -y nginx", "sudo systemctl enable nginx --now"]
}

# local-exec — runs on Terraform machine
provisioner "local-exec" {
  command = "ansible-playbook -i '${self.public_ip},' playbook.yml"
}

# local-exec on destroy
provisioner "local-exec" {
  when    = destroy
  command = "deregister.sh ${self.id}"
}

# null_resource with triggers (re-run on config change)
resource "null_resource" "configure" {
  triggers = { config_hash = filemd5("config.sh") }
  provisioner "remote-exec" {
    inline = ["bash /tmp/config.sh"]
  }
}

# Better alternative: user_data
resource "aws_instance" "web" {
  user_data = templatefile("bootstrap.sh.tpl", { env = var.environment })
}
```

---

> **Interview Takeaway:** Know how `remote-exec` works — SSH connection, connection block, inline/script/scripts forms. But the stronger answer is knowing when NOT to use it: HashiCorp considers provisioners a last resort because they break Terraform's state model, require network connectivity at apply time, and mix infrastructure provisioning with configuration management. For EC2, use `user_data` or Packer. Reserve `remote-exec` for platforms without a native cloud-init equivalent.
