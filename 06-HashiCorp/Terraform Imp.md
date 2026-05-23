# Terraform & Vault — Medium/Hard Interview Questions

> **Coverage:** HashiCorp Terraform internals, state management, module design, advanced HCL, Vault secrets engine deep-dives, auth methods, HA, and Terraform + Vault integration patterns
> **Level:** Mid-level (3–5 years) to Senior — questions that go beyond "what is Terraform"
> **Format:** Each question includes the core concept, deep explanation, code examples, real-world context, gotchas, and best practices

---

## Table of Contents

### Terraform Questions
- [Q1 — count vs for_each: When to use which and why for_each is almost always better](#q1)
- [Q2 — Lifecycle Meta-Arguments: create_before_destroy, prevent_destroy, ignore_changes, replace_triggered_by](#q2)
- [Q3 — Module Design: Inputs, outputs, versioning, and the flat vs nested module debate](#q3)
- [Q4 — Terraform Workspaces vs Separate State Files — which to use and when](#q4)
- [Q5 — Dynamic Blocks and Complex Variable Structures](#q5)
- [Q6 — terraform_remote_state vs Data Sources — tradeoffs and coupling risk](#q6)
- [Q7 — Terraform Import and the Import Block (1.5+) — bringing existing infra under management](#q7)
- [Q8 — Moved Blocks — safe refactoring without destroying resources](#q8)
- [Q9 — Provider Aliases, Version Constraints, and Multi-Region/Multi-Account Patterns](#q9)
- [Q10 — Sensitive Values, Output Security, and the `nonsensitive()` Function](#q10)
- [Q11 — terraform_data (1.4+) vs null_resource — why the replacement exists](#q11)
- [Q12 — Terraform Test Framework (1.6+) — writing unit and integration tests for modules](#q12)
- [Q13 — Backend Migration — safely moving state between backends without data loss](#q13)
- [Q14 — Dependency Inversion in Terraform — avoiding hidden coupling between modules](#q14)
- [Q15 — Terraform Plan Security — what `plan -out` exposes and how to protect it](#q15)

### Vault Questions
- [Q16 — Vault Token Hierarchy — root tokens, child tokens, orphan tokens, accessor tokens](#q16)
- [Q17 — Vault Lease System — TTL, renewal, revocation, and the "lease waterfall"](#q17)
- [Q18 — Vault PKI Secrets Engine — certificate lifecycle, intermediate CAs, auto-rotation](#q18)
- [Q19 — Vault Transit Secrets Engine — encryption as a service, key rotation, convergent encryption](#q19)
- [Q20 — Vault Agent and Vault Proxy — sidecar patterns, template rendering, auto-auth](#q20)
- [Q21 — Vault Namespaces — multi-tenancy, namespace isolation, hierarchical policies](#q21)
- [Q22 — Vault ACL Policy Design — path wildcards, capabilities, deny overrides, templating](#q22)
- [Q23 — Vault HA Clustering — Raft integrated storage vs Consul, leader election, performance standbys](#q23)
- [Q24 — Vault DR vs Performance Replication — which to use, RPO/RTO tradeoffs](#q24)
- [Q25 — Vault Response Wrapping — one-time tokens for secure secret delivery](#q25)
- [Q26 — Vault Entity and Identity — unified identity across auth methods, identity groups](#q26)
- [Q27 — Vault Audit Logging — mandatory audit devices, what gets logged, HMAC hashing](#q27)

### Combined Terraform + Vault
- [Q28 — Vault + Terraform at Scale: Secret injection patterns for multi-team, multi-env platforms](#q28)

---

<a name="q1"></a>
## Q1 — `count` vs `for_each`: When to use which and why `for_each` is almost always better

**The question:** *"When would you use `count` vs `for_each` in Terraform? What are the dangers of `count` with lists?"*

### The Core Difference

`count` creates resources indexed by integer position. `for_each` creates resources indexed by a stable string key.

```hcl
# count — resources identified by index [0], [1], [2]
resource "aws_iam_user" "dev" {
  count = length(var.usernames)
  name  = var.usernames[count.index]
}
# Creates: aws_iam_user.dev[0], aws_iam_user.dev[1], aws_iam_user.dev[2]

# for_each — resources identified by the value itself
resource "aws_iam_user" "dev" {
  for_each = toset(var.usernames)
  name     = each.value
}
# Creates: aws_iam_user.dev["alice"], aws_iam_user.dev["bob"], aws_iam_user.dev["charlie"]
```

### The count Trap — Index Shift Destroys Resources

```hcl
variable "usernames" {
  default = ["alice", "bob", "charlie"]
}
```

State after initial apply:
```
aws_iam_user.dev[0] → alice
aws_iam_user.dev[1] → bob
aws_iam_user.dev[2] → charlie
```

Now remove `"bob"` from the middle:
```hcl
variable "usernames" {
  default = ["alice", "charlie"]  # bob removed
}
```

Terraform plan shows:
```
~ aws_iam_user.dev[1]  # name: "bob" → "charlie"  (update)
- aws_iam_user.dev[2]  # destroyed entirely
```

Terraform destroys `charlie`'s user and renames `bob`'s user to `charlie`. Neither is what you wanted. With `for_each`, removing `"bob"` only destroys `aws_iam_user.dev["bob"]` — the others are untouched.

### When count IS Appropriate

```hcl
# count = 0 or 1 — the canonical conditional resource pattern
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? 1 : 0
  vpc   = true
}

# Output conditional resource
output "nat_ip" {
  value = var.enable_nat_gateway ? aws_eip.nat[0].public_ip : null
}

# Creating N identical replicas where order doesn't matter
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
}
```

### for_each with Maps — Richer Configurations

```hcl
variable "buckets" {
  type = map(object({
    versioning  = bool
    lifecycle   = string
    replication = bool
  }))
  default = {
    "logs"     = { versioning = false, lifecycle = "30d", replication = false }
    "backups"  = { versioning = true,  lifecycle = "90d", replication = true  }
    "artifacts"= { versioning = true,  lifecycle = "7d",  replication = false }
  }
}

resource "aws_s3_bucket" "app" {
  for_each = var.buckets
  bucket   = "${var.environment}-${each.key}"
}

resource "aws_s3_bucket_versioning" "app" {
  for_each = { for k, v in var.buckets : k => v if v.versioning }
  bucket   = aws_s3_bucket.app[each.key].id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**Rule:** Use `for_each` whenever you have named, distinct resources. Use `count` only for conditional (0/1) or truly interchangeable replicas.

---

<a name="q2"></a>
## Q2 — Lifecycle Meta-Arguments: `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`

**The question:** *"Walk me through Terraform's lifecycle block. When would you use each argument and what are the risks?"*

### create_before_destroy

Normally Terraform destroys the old resource, then creates the new one. `create_before_destroy = true` reverses this — create the replacement first, then destroy the old one.

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = "t3.medium"

  lifecycle {
    create_before_destroy = true
    # Why: ASGs reference this LT by ID.
    # If we destroy the LT first, the ASG has no valid LT until the new one is created.
    # create_before_destroy ensures there's always a valid LT.
  }
}
```

**Gotcha — naming conflicts:** If a resource uses a fixed name (not `name_prefix`), `create_before_destroy` fails because you can't have two resources with the same name simultaneously. Always use `name_prefix` for resources with this lifecycle.

```hcl
# ❌ This will fail — can't have two security groups named "web-sg" at once
resource "aws_security_group" "web" {
  name = "web-sg"                 # Fixed name → conflict on create_before_destroy
  lifecycle { create_before_destroy = true }
}

# ✅ Correct
resource "aws_security_group" "web" {
  name_prefix = "web-sg-"         # Unique name generated each time
  lifecycle { create_before_destroy = true }
}
```

### prevent_destroy

Protects production resources from accidental `terraform destroy` or destructive `terraform apply`.

```hcl
resource "aws_db_instance" "production" {
  identifier        = "prod-rds"
  engine            = "postgres"
  instance_class    = "db.r6g.large"
  allocated_storage = 100

  lifecycle {
    prevent_destroy = true
    # terraform destroy will error: "Instance cannot be destroyed"
    # Must manually set prevent_destroy = false and apply before destroying
  }
}
```

**Important:** `prevent_destroy` only works at plan/apply time. It does NOT protect against `terraform state rm` followed by a cloud console deletion.

### ignore_changes

Tells Terraform to ignore changes to specific attributes after initial creation.

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  lifecycle {
    ignore_changes = [
      ami,          # AMI updates managed by auto-scaling, not Terraform
      tags["LastDeployed"],  # Tag updated by external deployment system
      user_data,    # User data changes shouldn't force instance replacement
    ]
  }
}

# ignore all tags (when tags are managed by a separate tagging tool)
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  lifecycle {
    ignore_changes = [tags]
  }
}
```

**Gotcha:** `ignore_changes = all` is a code smell — it means Terraform is tracking a resource but never enforcing its configuration. Use sparingly and with a comment explaining why.

### replace_triggered_by (Terraform 1.2+)

Forces resource replacement when a *referenced* resource or attribute changes, even if the resource itself hasn't changed.

```hcl
resource "aws_launch_template" "app" {
  image_id = var.ami_id
  # ...
}

resource "aws_autoscaling_group" "app" {
  launch_template {
    id      = aws_launch_template.app.id
    version = aws_launch_template.app.latest_version
  }

  lifecycle {
    replace_triggered_by = [
      aws_launch_template.app.image_id
      # When the AMI changes → new LT version created →
      # ASG is replaced (triggering rolling instance refresh)
    ]
  }
}
```

This is useful for "rolling restart" patterns — replacing an ASG or ECS service when upstream config changes.

---

<a name="q3"></a>
## Q3 — Module Design: Inputs, Outputs, Versioning, and the Flat vs Nested Module Debate

**The question:** *"How do you design reusable Terraform modules? What makes a module well-designed vs poorly designed?"*

### Module Structure

```
modules/
└── ecs-service/
    ├── main.tf          # Resources
    ├── variables.tf     # Input variables with descriptions and validations
    ├── outputs.tf       # Outputs for consumers
    ├── versions.tf      # Required providers and Terraform version
    └── README.md        # Usage documentation
```

### Well-Designed Variables

```hcl
# variables.tf — strong typing, validation, sensible defaults
variable "service_name" {
  type        = string
  description = "Name of the ECS service. Used as prefix for all resources."

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,31}$", var.service_name))
    error_message = "Service name must be 3-32 chars, lowercase alphanumeric and hyphens, starting with a letter."
  }
}

variable "desired_count" {
  type        = number
  description = "Desired number of ECS task instances."
  default     = 2

  validation {
    condition     = var.desired_count >= 0 && var.desired_count <= 100
    error_message = "desired_count must be between 0 and 100."
  }
}

variable "container_definitions" {
  type = list(object({
    name      = string
    image     = string
    cpu       = number
    memory    = number
    port      = optional(number, null)
    env_vars  = optional(map(string), {})
    secrets   = optional(map(string), {})
  }))
  description = "List of container definitions for the ECS task."
}

variable "tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources."
  default     = {}
}
```

### Meaningful Outputs

```hcl
# outputs.tf — expose what callers actually need
output "service_arn" {
  description = "ARN of the ECS service."
  value       = aws_ecs_service.this.id
}

output "task_role_arn" {
  description = "ARN of the IAM role assigned to ECS tasks. Add policies to this role for AWS API access."
  value       = aws_iam_role.task.arn
}

output "security_group_id" {
  description = "ID of the ECS service security group. Use this to allow inbound from ALB."
  value       = aws_security_group.service.id
}

# Don't expose internal implementation details as outputs
# Bad: output "internal_target_group_arn" — callers shouldn't know TG details
```

### Module Versioning — Source and Version Pinning

```hcl
# Pin to a specific release tag — reproducible builds
module "ecs_service" {
  source  = "git::https://github.com/my-org/terraform-modules.git//ecs-service?ref=v2.3.1"

  service_name = "api"
  # ...
}

# Terraform Registry (public or private)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"   # Accept 5.x but not 6.x
}

# Version constraint operators:
# = 5.1.0   — exact version only
# != 5.0.0  — exclude this version
# > 5.0     — newer than 5.0
# >= 5.0, < 6.0  — range
# ~> 5.0    — 5.x (patch-level flexibility)
# ~> 5.1.0  — 5.1.x (patch-level only)
```

### Flat vs Nested Modules

**Flat (preferred):** Root module calls leaf modules directly. No modules calling other modules.

```
root/
  ├── main.tf (calls vpc module, ecs module, rds module)
  ├── modules/vpc/
  ├── modules/ecs-service/
  └── modules/rds/
```

**Nested (avoid unless necessary):** Module wraps other modules.

```
root/
  └── modules/
        └── full-stack/   ← calls vpc module AND ecs module inside it
              ├── main.tf
              └── modules/
                    ├── vpc/
                    └── ecs/
```

**Why flat is better:**
- Easier to understand dependency graph
- Easier to test modules independently
- Avoids provider inheritance bugs
- Forces consumers to compose explicitly rather than getting "magic" behavior

---

<a name="q4"></a>
## Q4 — Terraform Workspaces vs Separate State Files

**The question:** *"What's the difference between Terraform workspaces and separate directories with separate state files? When would you use workspaces?"*

### How Workspaces Work

```bash
# Workspaces share the same configuration directory
# but maintain separate state files

terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select dev
terraform apply   # → s3://bucket/env:/dev/terraform.tfstate

terraform workspace select prod
terraform apply   # → s3://bucket/env:/prod/terraform.tfstate
```

In code, `terraform.workspace` gives you the current workspace name:

```hcl
locals {
  environment = terraform.workspace   # "dev", "staging", "prod"

  instance_type = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "m6i.large"
  }[terraform.workspace]
}

resource "aws_instance" "app" {
  instance_type = local.instance_type
  tags = {
    Environment = local.environment
  }
}
```

### When Workspaces Are Appropriate

Workspaces work well when:
- **Environments are nearly identical** in configuration — same resources, just different sizes or counts
- **Short-lived environments** like per-PR ephemeral preview environments
- **Testing module changes** — spin up a workspace, test, destroy it

```bash
# PR preview environment pattern
terraform workspace new "pr-${PR_NUMBER}"
terraform apply -var="environment=pr-${PR_NUMBER}"
# ... PR tests run ...
terraform destroy
terraform workspace delete "pr-${PR_NUMBER}"
```

### When Separate Directories/State Are Better

Separate state files (different backends) are better for:
- **Production environments** that need strong isolation — a `terraform destroy` in `dev` with the wrong workspace selected could be catastrophic
- **Environments with meaningfully different architecture** — prod has multi-AZ RDS, dev has a single-instance
- **Separate access control** — dev team can't even read the prod state

```
environments/
├── dev/
│   ├── main.tf    → backend: s3://bucket/dev/terraform.tfstate
│   └── terraform.tfvars
├── staging/
│   ├── main.tf    → backend: s3://bucket/staging/terraform.tfstate
│   └── terraform.tfvars
└── prod/
    ├── main.tf    → backend: s3://bucket/prod/terraform.tfstate
    └── terraform.tfvars
```

**The key risk with workspaces:** You can accidentally apply to the wrong environment if you forget to switch workspaces. With separate directories, you physically must `cd` to the right directory.

| Factor | Workspaces | Separate Dirs/State |
|--------|-----------|---------------------|
| Configuration similarity | Must be very similar | Can differ completely |
| Isolation strength | Weak (same code, easy to misfire) | Strong (different dirs, different creds) |
| IAM/access control | Hard to differentiate | Easy — different IAM roles per env |
| Suitable for prod | No | Yes |
| Ephemeral environments | Yes — easy to create/destroy | Awkward |
| State visibility | One backend, multiple keys | Multiple backends |

---

<a name="q5"></a>
## Q5 — Dynamic Blocks and Complex Variable Structures

**The question:** *"How do you avoid repetitive HCL when a resource has a repeated nested block? Show me an example with dynamic blocks."*

### The Problem Without Dynamic Blocks

```hcl
# Repetitive and not data-driven
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

### With Dynamic Blocks

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    { port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"],  description = "HTTP" },
    { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"],  description = "HTTPS" },
    { port = 22,  protocol = "tcp", cidr_blocks = ["10.0.0.0/8"], description = "SSH from VPN" },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Conditional Dynamic Blocks

```hcl
variable "enable_encryption" {
  type    = bool
  default = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "bucket" {
  bucket = aws_s3_bucket.data.id

  rule {
    dynamic "apply_server_side_encryption_by_default" {
      for_each = var.enable_encryption ? [1] : []   # [1] = include block, [] = omit it
      content {
        sse_algorithm     = "aws:kms"
        kms_master_key_id = var.kms_key_arn
      }
    }
  }
}
```

### Nested Dynamic Blocks

```hcl
variable "policies" {
  type = list(object({
    name      = string
    effect    = string
    actions   = list(string)
    resources = list(string)
    conditions = optional(list(object({
      test     = string
      variable = string
      values   = list(string)
    })), [])
  }))
}

data "aws_iam_policy_document" "this" {
  dynamic "statement" {
    for_each = var.policies
    content {
      sid       = statement.value.name
      effect    = statement.value.effect
      actions   = statement.value.actions
      resources = statement.value.resources

      dynamic "condition" {
        for_each = statement.value.conditions
        content {
          test     = condition.value.test
          variable = condition.value.variable
          values   = condition.value.values
        }
      }
    }
  }
}
```

---

<a name="q6"></a>
## Q6 — `terraform_remote_state` vs Data Sources — Tradeoffs and Coupling Risk

**The question:** *"You have a networking layer and a compute layer in separate Terraform workspaces. How does the compute layer reference the VPC ID created by the networking layer? What are the tradeoffs?"*

### Option 1: `terraform_remote_state`

```hcl
# networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

# compute/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [aws_security_group.app.id]
  # ...
}
```

**Risk:** The compute layer now has read access to the *entire* networking state file — including any sensitive outputs. If the networking layer stores DB passwords or IAM keys in its state outputs, compute can read them.

### Option 2: Native Cloud Data Sources (Preferred)

```hcl
# compute/main.tf
# Query AWS directly — no coupling to Terraform state structure
data "aws_vpc" "main" {
  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = {
    Tier = "private"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.aws_subnets.private.ids[0]
  # ...
}
```

| Factor | `terraform_remote_state` | Native Data Sources |
|--------|--------------------------|---------------------|
| **Coupling** | Tight — must know exact output names | Loose — queries by tags/name |
| **Security** | Reads entire state (including sensitive values) | Reads only what you query |
| **Cross-cloud** | Works across providers | Provider-specific |
| **Non-Terraform resources** | Terraform-managed only | Any cloud resource |
| **Refactoring** | Breaking change if output name changes | Stable — based on cloud attributes |
| **Works without Terraform state access** | No | Yes — uses cloud API |

**Best practice:** Use native data sources for cross-layer references. Reserve `terraform_remote_state` for values that only exist in Terraform (like a random-generated database name that has no cloud API to query).

---

<a name="q7"></a>
## Q7 — Terraform Import and the Import Block (1.5+)

**The question:** *"You have 50 existing S3 buckets created manually. How do you bring them under Terraform management without deleting and recreating them?"*

### Old Method: CLI Import (Pre-1.5)

```bash
# Write the resource block first, then import one at a time
# terraform import <resource_address> <resource_id>

terraform import aws_s3_bucket.logs my-company-logs-bucket
terraform import aws_s3_bucket.data my-company-data-bucket
# Repeat 50 times...

# Problem: no plan preview, easy to make mistakes, tedious at scale
```

### New Method: Import Block (Terraform 1.5+)

```hcl
# import.tf — declare what to import
import {
  to = aws_s3_bucket.logs
  id = "my-company-logs-bucket"
}

import {
  to = aws_s3_bucket.data
  id = "my-company-data-bucket"
}

# Resource block (can be generated — see below)
resource "aws_s3_bucket" "logs" {
  bucket = "my-company-logs-bucket"
}
```

### Generate Configuration Automatically (Terraform 1.5+)

```bash
# Write the import block first, then let Terraform generate the resource config
terraform plan -generate-config-out=generated.tf

# generated.tf will contain:
resource "aws_s3_bucket" "logs" {
  bucket              = "my-company-logs-bucket"
  object_lock_enabled = false
  tags                = { "Environment" = "production" }
  # ... all current attributes
}

# Review generated config, clean it up, then:
terraform plan    # Should show "0 to add, 0 to change, 0 to destroy" if correct
terraform apply   # Writes import record to state
```

### Bulk Import with for_each

```hcl
# Import multiple buckets at once
locals {
  buckets = ["logs", "data", "backups", "artifacts"]
}

import {
  for_each = toset(local.buckets)
  to       = aws_s3_bucket.app[each.key]
  id       = "my-company-${each.key}"
}

resource "aws_s3_bucket" "app" {
  for_each = toset(local.buckets)
  bucket   = "my-company-${each.key}"
}
```

**After import:** Remove the `import {}` blocks once the apply succeeds — they're one-time declarations, not needed on subsequent applies.

---

<a name="q8"></a>
## Q8 — Moved Blocks — Safe Refactoring Without Destroying Resources

**The question:** *"You want to rename a resource in your Terraform config or move it into a module. How do you do this without Terraform destroying the old resource and creating a new one?"*

### The Problem

```hcl
# Before refactoring:
resource "aws_instance" "web_server" { ... }

# You rename it to:
resource "aws_instance" "app_server" { ... }

# Without moved block:
# terraform plan shows:
# - aws_instance.web_server  (destroy)
# + aws_instance.app_server  (create)
# ← This would terminate your production EC2 instance!
```

### The Solution: `moved` Block

```hcl
# Tell Terraform: "web_server" was renamed to "app_server" in state
moved {
  from = aws_instance.web_server
  to   = aws_instance.app_server
}

resource "aws_instance" "app_server" {
  # same config as before
}

# terraform plan now shows:
# ~ aws_instance.web_server → aws_instance.app_server (moved — no destroy)
```

### Moving Into a Module

```hcl
# Before: resource in root module
resource "aws_security_group" "web" { ... }

# After: moved into a module
module "web" {
  source = "./modules/web"
}

moved {
  from = aws_security_group.web
  to   = module.web.aws_security_group.this
}
```

### Moving Resources in for_each

```hcl
# Before: count-based
resource "aws_iam_user" "dev" {
  count = 3
  name  = ["alice", "bob", "charlie"][count.index]
}

# After: for_each-based (the right approach)
resource "aws_iam_user" "dev" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.value
}

# Moved blocks to map old indices to new keys
moved {
  from = aws_iam_user.dev[0]
  to   = aws_iam_user.dev["alice"]
}
moved {
  from = aws_iam_user.dev[1]
  to   = aws_iam_user.dev["bob"]
}
moved {
  from = aws_iam_user.dev[2]
  to   = aws_iam_user.dev["charlie"]
}
```

**Keep moved blocks:** Unlike import blocks, it's good practice to keep moved blocks in the codebase so that anyone applying an older state can still safely migrate. Remove them only after you're confident all state files have been updated.

---

<a name="q9"></a>
## Q9 — Provider Aliases, Version Constraints, and Multi-Region/Multi-Account Patterns

**The question:** *"How would you provision resources in multiple AWS regions and multiple AWS accounts from a single Terraform configuration?"*

### Provider Aliases — Multiple Regions

```hcl
# providers.tf
provider "aws" {
  region = "us-east-1"
  # Default provider — no alias needed for most resources
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_southeast_1"
  region = "ap-southeast-1"
}

# Use default provider (no alias) — resource in us-east-1
resource "aws_s3_bucket" "primary" {
  bucket = "my-bucket-primary"
}

# Use aliased provider — resource in eu-west-1
resource "aws_s3_bucket" "replica" {
  provider = aws.eu_west_1
  bucket   = "my-bucket-replica"
}
```

### Multi-Account Pattern — Assume Role

```hcl
# Deploying to multiple AWS accounts using IAM role assumption
provider "aws" {
  alias  = "shared_services"
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::111111111111:role/TerraformDeployRole"
    session_name = "terraform-shared-services"
  }
}

provider "aws" {
  alias  = "production"
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/TerraformDeployRole"
    session_name = "terraform-production"
  }
}

provider "aws" {
  alias  = "development"
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::333333333333:role/TerraformDeployRole"
    session_name = "terraform-development"
  }
}

# VPC peering between accounts — needs both providers
resource "aws_vpc_peering_connection" "shared_to_prod" {
  provider    = aws.shared_services
  vpc_id      = aws_vpc.shared.id
  peer_vpc_id = aws_vpc.prod.id
  peer_owner_id = "222222222222"
  auto_accept   = false
}

resource "aws_vpc_peering_connection_accepter" "prod_side" {
  provider                  = aws.production
  vpc_peering_connection_id = aws_vpc_peering_connection.shared_to_prod.id
  auto_accept               = true
}
```

### Provider Inheritance in Modules

```hcl
# Modules don't receive parent providers by default when using aliases
# You must explicitly pass the provider

module "ec2_replica" {
  source = "./modules/ec2"

  providers = {
    aws = aws.eu_west_1   # Pass the aliased provider to the module
  }
}

# Inside modules/ec2/main.tf — declare the expected provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
      configuration_aliases = [aws]  # Accept any aws provider config
    }
  }
}
```

---

<a name="q10"></a>
## Q10 — Sensitive Values, Output Security, and `nonsensitive()`

**The question:** *"How does Terraform handle sensitive data in variables, outputs, and state? What does `sensitive = true` actually protect?"*

### What `sensitive = true` Does and Doesn't Do

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

output "db_endpoint" {
  value = aws_db_instance.main.endpoint
}

output "db_password_output" {
  value     = aws_db_instance.main.password
  sensitive = true   # Terraform hides this in plan/apply output
}
```

```bash
terraform apply
# Outputs:
#   db_endpoint = "mydb.xxx.us-east-1.rds.amazonaws.com:5432"
#   db_password_output = <sensitive>   ← hidden from terminal
```

**What `sensitive = true` DOES NOT protect:**
- The value is stored in plaintext in `terraform.tfstate`
- The value appears in `terraform show` output (unless you also suppress that)
- The value is readable by anyone with state file access

```json
// terraform.tfstate — always stores values in plaintext
{
  "attributes": {
    "password": "MyActualPassword123!"  ← right here, readable
  }
}
```

### Sensitive Propagation — Taint by Association

```hcl
variable "api_key" {
  type      = string
  sensitive = true
}

# Any expression using a sensitive value becomes sensitive automatically
local "connection_string" {
  value = "postgres://user:${var.api_key}@host:5432/db"
  # This local is now automatically marked sensitive
}

output "conn_string" {
  value = local.connection_string
  # ↑ Terraform will error: "Output refers to sensitive values"
  # You must explicitly mark it sensitive:
  sensitive = true
}
```

### `nonsensitive()` — Use with Caution

```hcl
# Sometimes you need to use a sensitive value in a non-sensitive context
# (e.g., passing to a resource that doesn't support sensitive = true)

resource "local_file" "config" {
  content  = nonsensitive(var.api_key)   # Explicit: "I know this will be visible"
  filename = "/tmp/config.txt"
}

# Use only when you've verified the output won't be logged/stored insecurely
```

### Protecting Sensitive Outputs in CI

```bash
# Terraform plan stores sensitive values in the plan file
terraform plan -out=tfplan
# tfplan contains sensitive values in binary format — protect it

# Don't cat plan files in CI — sensitive values will be in logs
# ✅ Correct: show plan without values
terraform show -no-color tfplan | grep -v "sensitive"

# Store plan files encrypted if you must archive them
gpg --encrypt --recipient team@company.com tfplan
```

---

<a name="q11"></a>
## Q11 — `terraform_data` (1.4+) vs `null_resource`

**The question:** *"What is `null_resource` and why was `terraform_data` introduced in Terraform 1.4?"*

### null_resource — The Old Pattern

```hcl
# null_resource requires the "hashicorp/null" provider
terraform {
  required_providers {
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

resource "null_resource" "db_migration" {
  triggers = {
    migration_hash = filemd5("${path.module}/migrations/v2.sql")
    db_endpoint    = aws_db_instance.main.endpoint
  }

  provisioner "local-exec" {
    command = "flyway -url=jdbc:postgresql://${aws_db_instance.main.endpoint}/db migrate"
  }
}
```

### terraform_data — The Replacement (No Extra Provider)

```hcl
# terraform_data is built into Terraform 1.4+ — no provider needed
resource "terraform_data" "db_migration" {
  triggers_replace = [
    filemd5("${path.module}/migrations/v2.sql"),
    aws_db_instance.main.endpoint,
  ]

  provisioner "local-exec" {
    command = "flyway -url=jdbc:postgresql://${aws_db_instance.main.endpoint}/db migrate"
  }
}
```

### `terraform_data` as a Pure Data Store

`terraform_data` can also store arbitrary values in state:

```hcl
# Store computed values in state for later use
resource "terraform_data" "cluster_config" {
  input = {
    cluster_name = aws_eks_cluster.main.name
    endpoint     = aws_eks_cluster.main.endpoint
    ca_data      = aws_eks_cluster.main.certificate_authority[0].data
    created_at   = timestamp()
  }
}

output "cluster_info" {
  value = terraform_data.cluster_config.output
}
```

| Feature | `null_resource` | `terraform_data` |
|---------|----------------|-----------------|
| Provider required | Yes (`hashicorp/null`) | No (built-in) |
| Triggers field | `triggers = map` | `triggers_replace = list` |
| Stores values in state | No | Yes (`input`/`output`) |
| Available since | Terraform 0.x | Terraform 1.4 |
| Recommended | Legacy | Yes |

---

<a name="q12"></a>
## Q12 — Terraform Test Framework (1.6+)

**The question:** *"How do you test Terraform modules? What does the Terraform 1.6 test framework give you over tools like Terratest?"*

### Writing Tests with the Native Framework

```
modules/
└── s3-bucket/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── tests/
        ├── basic.tftest.hcl       # Unit-style: no real infra
        └── integration.tftest.hcl # Integration: creates real AWS resources
```

### Unit Test (No Infrastructure)

```hcl
# tests/basic.tftest.hcl — tests validation logic, locals, outputs
# without actually creating any AWS resources

variables {
  bucket_name = "test-bucket-12345"
  environment = "dev"
  versioning  = true
}

run "validate_bucket_name_format" {
  command = plan   # plan only — no AWS calls

  assert {
    condition     = aws_s3_bucket.this.bucket == "test-bucket-12345"
    error_message = "Bucket name should match the input variable"
  }

  assert {
    condition     = contains(keys(aws_s3_bucket.this.tags), "Environment")
    error_message = "Environment tag must be set"
  }
}

run "reject_invalid_environment" {
  command = plan

  variables {
    environment = "production"   # Invalid — only dev/staging/prod allowed
  }

  expect_failures = [
    var.environment   # This plan should fail on validation
  ]
}
```

### Integration Test (Real Infrastructure)

```hcl
# tests/integration.tftest.hcl — creates real resources, verifies, destroys

provider "aws" {
  region = "us-east-1"
}

variables {
  bucket_name = "tftest-integration-${run.setup.outputs.random_suffix}"
  environment = "dev"
}

run "setup" {
  module {
    source = "./tests/setup"   # Helper module that generates random suffix
  }
}

run "create_bucket" {
  command = apply   # Actually create the bucket

  assert {
    condition     = output.bucket_arn != ""
    error_message = "Bucket ARN should not be empty after creation"
  }
}

run "verify_versioning" {
  command = plan   # Re-plan to verify idempotency

  assert {
    condition     = aws_s3_bucket_versioning.this.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning should be enabled"
  }
}

# Resources are automatically destroyed after all runs complete
```

```bash
# Run tests
terraform test                                  # All tests
terraform test -filter=tests/basic.tftest.hcl  # Specific file
terraform test -verbose                         # Show all assertions
```

### Comparison with Terratest (Go)

| Factor | Native `terraform test` | Terratest (Go) |
|--------|------------------------|----------------|
| Language | HCL | Go |
| Setup complexity | None — built in | Go toolchain needed |
| Test expressiveness | Basic assertions | Full programming language |
| External API calls in tests | No | Yes (can call AWS API) |
| Parallel test execution | Limited | Full goroutine support |
| Best for | Module validation, unit-style checks | Complex integration tests, multi-module orchestration |

---

<a name="q13"></a>
## Q13 — Backend Migration — Moving State Between Backends

**The question:** *"Your team started using local state files and now needs to migrate to S3 + DynamoDB. How do you do this without losing state?"*

### Migration Steps

```hcl
# Step 1: Current state — local backend (default, no backend block)
# terraform.tfstate exists locally

# Step 2: Add the new backend configuration
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

```bash
# Step 3: Initialize — Terraform detects backend change and prompts
terraform init

# Output:
# Initializing the backend...
#
# Do you want to copy existing state to the new backend?
# Only 'yes' will be accepted to confirm.
#
# Enter a value: yes
#
# Successfully configured the backend "s3"!

# Step 4: Verify migration
terraform state list   # Should show all resources from local state
terraform plan         # Should show no changes
```

### Migrating Between Remote Backends

```bash
# S3 → Terraform Cloud migration
# Step 1: Pull current state
terraform state pull > migration-backup.tfstate

# Step 2: Update backend config to Terraform Cloud
# Step 3: Init with the new backend
terraform init

# When prompted: "yes" to migrate
# Terraform handles the copy automatically
```

### Cross-Environment State Copy (Manual)

```bash
# Scenario: Copy prod state to staging for testing
# Pull prod state
terraform -chdir=environments/prod state pull > prod-state.json

# Edit the state: change resource names, IDs as needed
# (advanced — usually not recommended, but sometimes necessary)

# Push to staging backend
terraform -chdir=environments/staging state push prod-state.json
```

---

<a name="q14"></a>
## Q14 — Dependency Inversion in Terraform — Avoiding Hidden Coupling

**The question:** *"How do you design Terraform configurations to minimize tight coupling between teams and layers?"*

### The Problem: Implicit Coupling

```hcl
# BAD: compute module hard-codes knowledge of the networking module's internals
module "compute" {
  source = "./modules/compute"

  # Directly referencing networking module outputs — creates coupling
  vpc_id    = module.networking.vpc_id
  subnet_id = module.networking.private_subnet_ids[0]

  # If networking module renames "private_subnet_ids" → compute breaks
}
```

### Interface-Based Design

```hcl
# GOOD: compute module receives generic interface (just an ID)
# It doesn't need to know where the subnet ID came from

variable "subnet_id" {
  type        = string
  description = "ID of the subnet to place compute resources in."
  validation {
    condition     = can(regex("^subnet-[a-f0-9]+$", var.subnet_id))
    error_message = "Must be a valid AWS subnet ID."
  }
}

# The caller decides where the ID comes from:
module "compute" {
  source    = "./modules/compute"
  subnet_id = data.aws_subnet.private.id   # Could be from anywhere
}
```

### Using SSM Parameter Store as the Interface Contract

```hcl
# Networking layer publishes IDs to SSM — the "API" between layers
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/${var.environment}/networking/vpc-id"
  type  = "String"
  value = aws_vpc.main.id
}

resource "aws_ssm_parameter" "private_subnets" {
  name  = "/${var.environment}/networking/private-subnet-ids"
  type  = "StringList"
  value = join(",", aws_subnet.private[*].id)
}

# Compute layer reads from SSM — no Terraform state dependency
data "aws_ssm_parameter" "vpc_id" {
  name = "/${var.environment}/networking/vpc-id"
}

data "aws_ssm_parameter" "private_subnets" {
  name = "/${var.environment}/networking/private-subnet-ids"
}
```

This pattern means:
- The compute team doesn't need state access to the networking state
- The networking team can refactor internally without breaking compute
- The SSM path is the stable contract between layers

---

<a name="q15"></a>
## Q15 — Terraform Plan Security — What `-out` Exposes

**The question:** *"A CI pipeline saves the terraform plan output and applies it in a separate job. What security risks does this introduce?"*

### The Risk: Plan Files Contain Secrets

```bash
terraform plan -out=tfplan   # Saves a binary plan file
terraform apply tfplan       # Applies exactly what was planned

# The tfplan file contains:
# - All sensitive variable values (db passwords, API keys)
# - Proposed state changes (including resource attributes)
# - Provider credentials snapshot
# In a binary format — but trivially readable:

terraform show tfplan        # Shows full plan including sensitive values
# password = "MyActualSecret!"   ← visible in clear text
```

### Attack Scenario: Plan File Interception

```
CI Pipeline:
  Job 1: terraform plan -out=tfplan → uploads tfplan as artifact
  Job 2: downloads tfplan → terraform apply tfplan

Attacker with read access to CI artifacts:
  Downloads tfplan → terraform show tfplan → reads all secrets
```

### Mitigations

```bash
# 1. Encrypt plan files before storing as artifacts
# In GitHub Actions:
- name: Encrypt plan
  run: |
    gpg --batch --yes --passphrase "${{ secrets.GPG_PASSPHRASE }}" \
      --symmetric --cipher-algo AES256 tfplan
    rm tfplan

- name: Upload encrypted plan
  uses: actions/upload-artifact@v4
  with:
    name: tfplan-encrypted
    path: tfplan.gpg

# 2. Use Terraform Cloud — plan and apply run in the same secure context
# Plan files never leave the TFC environment

# 3. Short-lived plan files — don't store long-term
# Delete after apply regardless of success/failure
- name: Apply
  run: |
    terraform apply tfplan
  if: always()
- name: Cleanup
  run: rm -f tfplan
  if: always()

# 4. Use OIDC for cloud auth — no long-lived credentials in plan files
provider "aws" {
  # OIDC-based auth — no AWS_ACCESS_KEY_ID/SECRET_ACCESS_KEY stored
}
```

---

<a name="q16"></a>
## Q16 — Vault Token Hierarchy: Root Tokens, Child Tokens, Orphan Tokens

**The question:** *"Explain Vault's token model. What's the difference between a root token, a service token, and a batch token? What is token hierarchy and why does it matter for revocation?"*

### Token Types

```bash
# Root token — created during vault operator init
# Has unlimited capabilities — should be revoked immediately after setup
vault token lookup
# policies: ["root"]
# Never use in automation — create a non-root token with specific policies

# Service token — the default, persists in storage
vault token create -policy="my-policy" -ttl=1h
# token: s.xxxxxxxxxxxxxxxxxxxxxxxx
# Has a lease, can be renewed, parent-child relationship

# Batch token — lightweight, not persisted (Vault 1.0+)
vault token create -type=batch -policy="my-policy" -ttl=1h
# token: b.AAAAAQ...  (starts with b.)
# Faster (no storage write), but cannot be renewed, no accessor
# Best for high-throughput microservices with short-lived requests
```

### Token Hierarchy and Revocation Cascade

```
Root Token
  ↓ creates
  Service Token A (TTL: 24h)
    ↓ creates
    Service Token B (TTL: 1h)  ← your CI pipeline token
      ↓ uses to create
      Dynamic DB Credential (TTL: 30m)
      Dynamic DB Credential (TTL: 30m)
```

When Token A is revoked, **all descendants are revoked automatically** — Token B and all its dynamic credentials are immediately invalidated. This is called **hierarchical revocation**.

### Orphan Tokens — Breaking the Hierarchy

```bash
# Orphan token has no parent — revocation of creator doesn't affect it
vault token create -policy="my-policy" -orphan -ttl=24h
# Use for long-running processes that should survive admin token rotation
# Use for Vault Agent — it needs its token to outlive the agent's creator
```

### Accessor Tokens — Lookup Without the Token

```bash
# Every token has an accessor — a way to inspect/revoke without knowing the token
vault token create -policy="my-policy" -ttl=1h
# token_accessor: abc123

# Look up a token by accessor (without the token itself)
vault token lookup -accessor abc123

# Revoke by accessor
vault token revoke -accessor abc123

# Use case: audit who has tokens without exposing the tokens themselves
vault list auth/token/accessors   # List all active token accessors
```

---

<a name="q17"></a>
## Q17 — Vault Lease System: TTL, Renewal, and Revocation

**The question:** *"Explain Vault's lease system. What happens when a dynamic credential's TTL expires? How does an application renew its lease?"*

### Lease Lifecycle

```
vault read database/creds/my-role
→ Response:
  lease_id:       database/creds/my-role/abc123xyz
  lease_duration: 1h
  renewable:      true
  data:
    username: v-token-myapp-xKj3Lm
    password: A1b2C3-dynamically-generated

After 1h: Vault revokes the lease
  → Vault executes the revocation SQL:
    REVOKE 'v-token-myapp-xKj3Lm'@'%'
  → The database user is automatically deleted
```

### Lease Renewal

```bash
# Renew before expiry — extends TTL up to max_ttl
vault lease renew database/creds/my-role/abc123xyz
# lease_duration: 1h  (renewed for another hour)

# Check remaining time
vault lease lookup database/creds/my-role/abc123xyz
# expire_time: 2024-01-15T12:00:00Z
```

### Application-Side Renewal (Vault SDK)

```python
import hvac
import time
import threading

client = hvac.Client(url="https://vault.example.com")
client.token = os.environ["VAULT_TOKEN"]

# Get dynamic credential
creds = client.secrets.database.get_credentials(name="my-role")
lease_id = creds["lease_id"]
ttl = creds["lease_duration"]  # seconds

# Renew the lease at 80% of TTL
def renew_lease():
    while True:
        time.sleep(ttl * 0.8)
        try:
            result = client.sys.renew_self(lease_id=lease_id)
            ttl = result["lease_duration"]
        except Exception as e:
            # Lease expired or revoked — re-fetch credentials
            creds = client.secrets.database.get_credentials(name="my-role")
            lease_id = creds["lease_id"]

thread = threading.Thread(target=renew_lease, daemon=True)
thread.start()
```

### Lease Revocation — Immediate vs Prefix

```bash
# Revoke a specific lease immediately
vault lease revoke database/creds/my-role/abc123xyz

# Revoke ALL leases for a role (incident response)
vault lease revoke -prefix database/creds/my-role

# Revoke all leases from a specific auth mount
vault lease revoke -prefix auth/kubernetes/
```

**Max TTL:** Even with renewals, a lease cannot be extended beyond `max_ttl`. If a credential reaches max_ttl, the application must re-authenticate and get a new lease.

---

<a name="q18"></a>
## Q18 — Vault PKI Secrets Engine: Certificate Lifecycle and Auto-Rotation

**The question:** *"How would you use Vault to manage TLS certificates for a microservices mesh? What's the difference between a root CA and an intermediate CA in Vault?"*

### PKI Architecture: Root → Intermediate → Leaf

```
Root CA (Vault or offline HSM)
  ↑ keeps private key offline — only used to sign the intermediate
  |
  └─→ Intermediate CA (Vault-hosted)
         ↑ actively used to sign leaf certificates
         |
         └─→ Leaf Certificate (service-specific, short TTL)
                 e.g., api.prod.internal, worker.prod.internal
```

### Setting Up PKI Engine

```bash
# 1. Enable PKI engine for root CA
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki  # 10 years for root

# 2. Generate root CA (keep private key inside Vault)
vault write pki/root/generate/internal \
  common_name="My Company Root CA" \
  ttl=87600h

# 3. Enable PKI engine for intermediate CA
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int  # 5 years for intermediate

# 4. Generate intermediate CSR
vault write pki_int/intermediate/generate/internal \
  common_name="My Company Intermediate CA" \
  > /tmp/pki_intermediate.csr

# 5. Sign intermediate with root CA
vault write pki/root/sign-intermediate \
  csr=@/tmp/pki_intermediate.csr \
  format=pem_bundle \
  ttl="43800h" \
  > /tmp/intermediate.cert.pem

# 6. Import signed intermediate certificate
vault write pki_int/intermediate/set-signed \
  certificate=@/tmp/intermediate.cert.pem
```

### Creating Roles and Issuing Certificates

```bash
# Create a role that can issue certs for *.prod.internal
vault write pki_int/roles/prod-dot-internal \
  allowed_domains="prod.internal" \
  allow_subdomains=true \
  max_ttl="72h"            # Short TTL — rotate frequently, not forever

# Issue a certificate for a service
vault write pki_int/issue/prod-dot-internal \
  common_name="api.prod.internal" \
  alt_names="api-v2.prod.internal" \
  ttl="24h"
```

### Auto-Rotation with Vault Agent

```hcl
# vault-agent-config.hcl — automatically renew certs before expiry
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config {
      role = "my-app-role"
    }
  }
}

template {
  source      = "/etc/vault-agent/tls.tpl"
  destination = "/etc/ssl/certs/service.crt"
  command     = "systemctl reload nginx"  # Reload after cert rotation
  
  # Renew when 10% of TTL remains (e.g., 2.4 hours before expiry for a 24h cert)
  error_on_missing_key = true
}
```

```
# tls.tpl — Consul Template format
{{ with secret "pki_int/issue/prod-dot-internal" "common_name=api.prod.internal" "ttl=24h" }}
{{ .Data.certificate }}
{{ .Data.issuing_ca }}
{{ end }}
```

---

<a name="q19"></a>
## Q19 — Vault Transit Secrets Engine: Encryption as a Service

**The question:** *"What is the Vault Transit engine and how is it different from just storing secrets in KV?"*

### KV vs Transit — The Conceptual Difference

- **KV:** Vault stores your secret. Your app gives Vault the plaintext; Vault keeps it.
- **Transit:** Vault never stores your data. Your app gives Vault data to encrypt; Vault returns ciphertext. Your app stores the ciphertext. Vault holds only the key.

```
KV:    App → (secret) → Vault stores it → App retrieves it
Transit: App → (plaintext) → Vault encrypts → (ciphertext) → App stores it in its own DB
```

### Transit in Practice

```bash
# Enable transit engine
vault secrets enable transit

# Create an encryption key
vault write -f transit/keys/payments
# key_type: aes256-gcm96 (default)
# deletion_allowed: false

# Encrypt data
vault write transit/encrypt/payments \
  plaintext=$(echo "4111111111111111" | base64)
# ciphertext: vault:v1:abc123...

# Decrypt data
vault write transit/decrypt/payments \
  ciphertext="vault:v1:abc123..."
# plaintext: NDExMTExMTExMTExMTExMQ==  (base64 of card number)

echo "NDExMTExMTExMTExMTExMQ==" | base64 --decode
# 4111111111111111
```

### Key Rotation — Zero Downtime

```bash
# Rotate the encryption key (creates version 2)
vault write -f transit/keys/payments/rotate
# Keys are now: v1 (old), v2 (current)

# All NEW encryptions use v2
# All EXISTING ciphertexts (vault:v1:...) are still decryptable with v1

# Rewrap all old ciphertexts to use the new key
# (without ever exposing plaintext to your app)
vault write transit/rewrap/payments \
  ciphertext="vault:v1:abc123..."
# Returns: vault:v2:xyz789...   ← same plaintext, new key version

# After rewrapping all ciphertexts, disable v1 (prevent decryption of old ciphertexts)
vault write transit/keys/payments/config \
  min_decryption_version=2   # v1 can no longer decrypt
```

### Application Integration

```python
import base64
import hvac

vault = hvac.Client(url="https://vault.example.com", token=os.environ["VAULT_TOKEN"])

def encrypt_card_number(card_number: str) -> str:
    encoded = base64.b64encode(card_number.encode()).decode()
    result = vault.secrets.transit.encrypt_data(
        name="payments",
        plaintext=encoded
    )
    return result["data"]["ciphertext"]  # Store this in your database

def decrypt_card_number(ciphertext: str) -> str:
    result = vault.secrets.transit.decrypt_data(
        name="payments",
        ciphertext=ciphertext
    )
    return base64.b64decode(result["data"]["plaintext"]).decode()
```

**PCI-DSS use case:** Never store raw card numbers — store Transit ciphertext. Only apps with Transit `decrypt` policy can read the real number. Key rotation happens in Vault without changing the database.

---

<a name="q20"></a>
## Q20 — Vault Agent and Vault Proxy: Sidecar Patterns and Auto-Auth

**The question:** *"Your application containers on Kubernetes need to access Vault secrets. What is Vault Agent and how does it simplify secret injection?"*

### The Problem Without Vault Agent

Without an agent, each application must:
1. Authenticate to Vault (Kubernetes auth, manage service account tokens)
2. Fetch secrets from Vault
3. Handle token renewal
4. Re-fetch secrets before they expire
5. Write secrets to config files or inject into environment

Every app reinvents this wheel.

### Vault Agent — Sidecar That Does It All

```
Pod
├── app container        — reads /vault/secrets/db-creds
└── vault-agent sidecar  — handles auth, fetches secrets, writes files, renews
```

```hcl
# vault-agent-config.hcl
vault {
  address = "https://vault.company.com"
}

auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config {
      role = "my-app-role"   # Vault K8s role mapped to this service account
    }
  }

  sink "file" {
    config {
      path = "/home/vault/.vault-token"
    }
  }
}

template {
  source      = "/vault/templates/db-creds.ctmpl"
  destination = "/vault/secrets/db-creds"
  # File is written before app container starts (init container mode)
  # Kept up to date when secrets rotate
}
```

```
# db-creds.ctmpl — Consul Template syntax
{{ with secret "database/creds/my-role" }}
DB_USER={{ .Data.username }}
DB_PASS={{ .Data.password }}
DB_HOST=mydb.internal:5432
{{ end }}
```

### Kubernetes Deployment

```yaml
# kubernetes/deployment.yaml
spec:
  serviceAccountName: my-app-sa   # K8s SA bound to Vault role
  
  initContainers:
  - name: vault-agent-init
    image: hashicorp/vault:1.15
    args: ["agent", "-config=/vault/config/config.hcl", "-exit-after-auth"]
    # Runs once: authenticates, fetches secrets, writes files, exits
    volumeMounts:
    - name: vault-config
      mountPath: /vault/config
    - name: vault-secrets
      mountPath: /vault/secrets

  containers:
  - name: app
    image: my-app:latest
    env:
    - name: DB_CREDS_FILE
      value: /vault/secrets/db-creds
    volumeMounts:
    - name: vault-secrets
      mountPath: /vault/secrets

  - name: vault-agent  # Sidecar for ongoing renewal and rotation
    image: hashicorp/vault:1.15
    args: ["agent", "-config=/vault/config/config.hcl"]
    volumeMounts:
    - name: vault-config
      mountPath: /vault/config
    - name: vault-secrets
      mountPath: /vault/secrets
```

### Vault Proxy (1.13+) — Forward Proxy Mode

Vault Proxy caches Vault API responses locally, reducing latency and load on Vault:

```hcl
# vault-proxy-config.hcl
api_proxy {
  use_auto_auth_token = "force"
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = true
}

# App talks to localhost:8200 instead of vault.company.com
# Proxy handles auth and caches responses
cache {
  use_auto_auth_token = true
}
```

---

<a name="q21"></a>
## Q21 — Vault Namespaces: Multi-Tenancy and Hierarchical Policies

**The question:** *"Your company has 3 business units each needing their own Vault environment. How do you isolate them without running 3 separate Vault clusters?"*

### Vault Namespaces (Enterprise Feature)

Namespaces create isolated tenants within a single Vault cluster. Each namespace has its own:
- Auth methods
- Secret engines
- Policies
- Tokens (scoped to their namespace)

```bash
# Root namespace creates child namespaces
vault namespace create engineering
vault namespace create finance
vault namespace create hr

# Work within a namespace — two methods:
# Method 1: VAULT_NAMESPACE env var
export VAULT_NAMESPACE=engineering
vault secrets enable kv-v2
vault kv put secret/db username=admin password=secret

# Method 2: -namespace flag per command
vault -namespace=finance secrets enable kv-v2
vault -namespace=finance kv put secret/payments api-key=xyz

# Nested namespaces (enterprise)
vault namespace create -namespace=engineering platform-team
# Namespace: engineering/platform-team
```

### Namespace Isolation

```bash
# Token created in "engineering" namespace cannot access "finance" namespace
export VAULT_NAMESPACE=engineering
vault token create -policy=default

# That token:
# ✅ Can read engineering/* secrets
# ❌ Cannot read finance/* secrets
# ❌ Cannot even see that finance namespace exists
```

### Cross-Namespace Communication

```hcl
# Vault policy in "engineering" allowing access to a secret in root namespace
path "sys/namespaces/*" {
  capabilities = ["read"]
}

# Engineers can read a shared secret from root namespace
path "secret/data/shared-certificates" {
  capabilities = ["read"]
}
```

---

<a name="q22"></a>
## Q22 — Vault ACL Policy Design: Path Wildcards, Capabilities, Deny, Templating

**The question:** *"Write a Vault policy for a CI pipeline that can only read database credentials for its own environment. How do you prevent dev from reading prod secrets?"*

### Capabilities Reference

```hcl
# All Vault capabilities:
# create  — PUT/POST new data (not update)
# read    — GET data
# update  — POST existing data (update)
# delete  — DELETE data
# list    — LIST keys (directory listing)
# patch   — PATCH partial update (Vault 1.9+)
# sudo    — Access privileged paths (root-protected)
# deny    — Explicitly deny — overrides everything
```

### Environment-Scoped Policy with Templating

```hcl
# ci-pipeline-policy.hcl
# Policy uses identity metadata to scope access to the pipeline's own environment

# Read database credentials only for this environment
path "database/creds/{{identity.entity.metadata.environment}}-*" {
  capabilities = ["read"]
}

# Read KV secrets for this environment only
path "secret/data/{{identity.entity.metadata.environment}}/*" {
  capabilities = ["read"]
}

# List available credentials (but not read other envs)
path "secret/metadata/{{identity.entity.metadata.environment}}/*" {
  capabilities = ["list"]
}

# Explicitly deny prod access from non-prod identities
path "secret/data/prod/*" {
  capabilities = ["deny"]
}

# Allow token self-renewal
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Allow token self-lookup
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
```

```bash
# Create entity for the dev pipeline with environment metadata
vault write identity/entity \
  name="ci-pipeline-dev" \
  metadata=environment=dev \
  policies="ci-pipeline-policy"

# Create entity for prod pipeline
vault write identity/entity \
  name="ci-pipeline-prod" \
  metadata=environment=prod \
  policies="ci-pipeline-policy"

# Now the SAME policy gives different access based on metadata:
# ci-pipeline-dev: can read database/creds/dev-* only
# ci-pipeline-prod: can read database/creds/prod-* only
```

### Deny Always Wins

```hcl
# If multiple policies apply to a token:
# Policy A: allow read on secret/data/prod/*
# Policy B: deny read on secret/data/prod/*
# → DENY wins — the token cannot read prod secrets

# Deny is the safety net — use it explicitly for cross-environment protection
```

---

<a name="q23"></a>
## Q23 — Vault HA Clustering: Raft vs Consul, Leader Election, Performance Standbys

**The question:** *"How would you set up a highly available Vault cluster? What's the difference between using Raft integrated storage and Consul as a backend?"*

### Raft Integrated Storage (Recommended Since Vault 1.4)

```hcl
# vault-config.hcl — Raft integrated storage
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"

  retry_join {
    leader_api_addr = "https://vault-node-2.internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-node-3.internal:8200"
  }
}

listener "tcp" {
  address            = "0.0.0.0:8200"
  cluster_address    = "0.0.0.0:8201"
  tls_cert_file      = "/etc/vault/tls/vault.crt"
  tls_key_file       = "/etc/vault/tls/vault.key"
}

api_addr     = "https://vault-node-1.internal:8200"
cluster_addr = "https://vault-node-1.internal:8201"
```

```bash
# Initialize the cluster (run once on first node)
vault operator init -key-shares=5 -key-threshold=3

# Join other nodes to the cluster
vault operator raft join https://vault-node-1.internal:8200

# Check cluster status
vault operator raft list-peers
# Node ID    Address                      State    Voter
# vault-1    vault-node-1.internal:8201   leader   true
# vault-2    vault-node-2.internal:8201   follower true
# vault-3    vault-node-3.internal:8201   follower true
```

### Consul Backend (Legacy Approach)

```hcl
storage "consul" {
  address = "consul.internal:8500"
  path    = "vault/"
  token   = "consul-token"
}
```

| Factor | Raft Integrated | Consul Backend |
|--------|----------------|----------------|
| External dependency | None | Requires Consul cluster |
| Operational complexity | Low (Vault manages its own consensus) | High (maintain two distributed systems) |
| Performance | Faster reads (local storage) | Slower (Consul network round-trips) |
| Consistency | Strong (Raft consensus) | Strong (Consul consensus) |
| Vault recommendation | ✅ Preferred | Legacy, not recommended for new installs |

### Performance Standbys (Enterprise)

```
Vault Cluster:
  Active Node (leader)    — reads AND writes
  Performance Standby 1   — reads only (no writes)
  Performance Standby 2   — reads only (no writes)

Write request → active node
Read request  → any node (load-balanced)
```

Performance standbys serve read traffic directly without forwarding to the leader — critical for high-read workloads like PKI certificate issuance.

---

<a name="q24"></a>
## Q24 — Vault DR vs Performance Replication

**The question:** *"What's the difference between Vault DR replication and Performance replication? When would you use each?"*

### Disaster Recovery (DR) Replication

```
Primary Cluster (us-east-1)  →  DR Secondary (us-west-2)
- Replicates: all data, policies, tokens
- DR secondary: PASSIVE — cannot serve requests
- Promotion: manual (operator initiates failover)
- Purpose: survive region failure
```

```bash
# Promote DR secondary to primary during a disaster
vault operator generate-root -dr-token   # Generate recovery token
vault operator raft snapshot restore     # If needed
# DR secondary becomes new primary
# All tokens from original primary continue to work
```

**RPO:** Near-zero (continuous replication)
**RTO:** Minutes (manual promotion process)

### Performance Replication (Enterprise)

```
Primary Cluster (us-east-1)  →  Performance Secondary (eu-west-1)
- Replicates: policies and configuration (NOT tokens)
- Secondary: ACTIVE — can serve read and write requests
- Purpose: serve local traffic, reduce latency for global teams
- Write requests: forwarded to primary cluster

Local team in EU → vault.eu.company.com (performance secondary)
  → Read secrets: served locally (low latency)
  → Write secrets: forwarded to us-east-1 primary (higher latency)
```

| Factor | DR Replication | Performance Replication |
|--------|---------------|------------------------|
| Secondary can serve traffic | No | Yes |
| Tokens replicated | Yes | No (re-auth required on secondary) |
| Purpose | Disaster recovery | Geo-distributed access, latency reduction |
| Failover | Manual | Automatic for reads |
| License requirement | Enterprise | Enterprise |

---

<a name="q25"></a>
## Q25 — Vault Response Wrapping: Secure Secret Delivery

**The question:** *"How would you securely deliver a Vault secret to an application at bootstrap without the secret ever being readable by an intermediary (like CI)?"*

### The Problem: Secret Delivery

```
Scenario: CI pipeline needs to give a new EC2 instance its initial Vault credentials
Problem:  The pipeline knows the credentials → credentials appear in CI logs → insecure
```

### Response Wrapping — One-Time Token

```bash
# CI pipeline creates a wrapped token — it can ONLY be unwrapped ONCE
# CI never sees the actual secret

# Step 1: CI creates a response-wrapped token for the new instance
WRAP_TOKEN=$(vault token create \
  -policy="app-policy" \
  -ttl=1h \
  -wrap-ttl=15m \   # Wrapping token expires in 15 minutes if not used
  -field=wrapping_token)

# WRAP_TOKEN is a one-time-use token — it can only retrieve the real token ONCE
# If intercepted and used, the second use attempt reveals the interception

# Step 2: Pass WRAP_TOKEN to the instance (e.g., via EC2 user data or tags)

# Step 3: Instance unwraps the token to get the real token
REAL_TOKEN=$(vault unwrap -field=token $WRAP_TOKEN)

# Step 4: Instance uses REAL_TOKEN for all subsequent Vault requests
```

### Detecting Token Interception

```bash
# If the wrapping token was already used when your app tries to unwrap:
vault unwrap $WRAP_TOKEN
# Error: wrapping token is not valid or does not exist

# This is a security alert: the token was intercepted and used by someone else
# Vault emits an audit log event for failed unwrap attempts
# Trigger incident response: the secret was potentially compromised
```

### Implementation Pattern for EC2 Bootstrap

```hcl
# Terraform provisions the instance and creates a wrapped token
data "vault_generic_secret" "wrapped_approle" {
  path = "auth/approle/role/app/secret-id"

  # Response wrapping — terraform gets a wrap token, not the actual secret ID
  # (using vault_wrapping_lookup data source pattern)
}

resource "aws_instance" "app" {
  user_data = base64encode(templatefile("bootstrap.sh.tpl", {
    wrap_token = data.vault_generic_secret.wrapped_approle.data["wrapping_token"]
    vault_addr = "https://vault.company.com"
    role_id    = vault_approle_auth_backend_role_id.app.role_id
  }))
}
```

```bash
# bootstrap.sh.tpl — runs on first boot
#!/bin/bash
# Unwrap the secret ID — one-time operation
SECRET_ID=$(VAULT_ADDR="${vault_addr}" vault unwrap -field=secret_id ${wrap_token})

# Authenticate with AppRole using role_id + secret_id
export VAULT_TOKEN=$(vault write -field=token auth/approle/login \
  role_id="${role_id}" secret_id="$SECRET_ID")

# Destroy the variable — no longer needed
unset SECRET_ID

# Now use VAULT_TOKEN for all subsequent operations
```

---

<a name="q26"></a>
## Q26 — Vault Entity and Identity: Unified Identity Across Auth Methods

**The question:** *"A user authenticates to Vault sometimes via LDAP and sometimes via GitHub. How do Vault entities and groups work to give them consistent access?"*

### The Problem Without Entities

```
User "alice":
  Auth via LDAP:    → alice-ldap token (policies from LDAP config)
  Auth via GitHub:  → alice-github token (policies from GitHub config)

These are TWO separate identities in Vault. No shared attributes, no unified policy.
```

### Vault Identity System

```bash
# Entities represent real-world identities (users, services)
# Aliases map auth method identities to the entity

# Create entity for Alice
vault write identity/entity \
  name="alice" \
  policies="developer-policy" \
  metadata="department=engineering" \
  metadata="team=platform"

ENTITY_ID=$(vault read -field=id identity/entity/name/alice)

# Create alias: "alice-ldap" on LDAP auth → Alice entity
vault write identity/entity-alias \
  name="alice@company.com" \
  canonical_id=$ENTITY_ID \
  mount_accessor=$(vault auth list -format=json | jq -r '."ldap/".accessor')

# Create alias: "alice" on GitHub auth → same Alice entity
vault write identity/entity-alias \
  name="alice" \
  canonical_id=$ENTITY_ID \
  mount_accessor=$(vault auth list -format=json | jq -r '."github/".accessor')

# Now regardless of how Alice authenticates, she gets:
# - The entity's policies (developer-policy)
# - Her LDAP/GitHub auth method's policies
# - Her metadata (department, team) — usable in policy templating
```

### Identity Groups — Team-Based Access

```bash
# Internal group: members managed by Vault operators
vault write identity/group \
  name="platform-team" \
  policies="platform-policy,infrastructure-policy" \
  member_entity_ids="$ALICE_ID,$BOB_ID,$CHARLIE_ID"

# External group: membership driven by auth method (LDAP group sync)
vault write identity/group \
  name="ldap-engineers" \
  type="external" \
  policies="engineer-policy"

# Create alias connecting LDAP group "engineers" → Vault group "ldap-engineers"
vault write identity/group-alias \
  name="engineers" \
  canonical_id=$(vault read -field=id identity/group/name/ldap-engineers) \
  mount_accessor=$(vault auth list -format=json | jq -r '."ldap/".accessor')

# Now anyone in LDAP "engineers" group automatically gets engineer-policy in Vault
# When they're removed from LDAP group, they lose access on next login
```

---

<a name="q27"></a>
## Q27 — Vault Audit Logging: What Gets Logged, HMAC Hashing

**The question:** *"How does Vault's audit system work? What does an audit log entry look like and why are sensitive values hashed?"*

### Audit Devices

```bash
# Enable multiple audit devices — ALL must succeed for a request to complete
vault audit enable file file_path=/var/log/vault/audit.log
vault audit enable syslog tag="vault" facility="AUTH"

# If ANY enabled audit device fails (disk full, syslog down):
# Vault BLOCKS the request — "fail-closed" behavior
# This prevents requests from being processed without an audit trail

# List enabled audit devices
vault audit list
```

### Anatomy of an Audit Log Entry

```json
{
  "time": "2024-01-15T10:00:00.000000Z",
  "type": "request",
  "auth": {
    "client_token": "hmac-sha256:abc123...",  // Token hashed, never logged in clear
    "accessor": "hmac-sha256:def456...",
    "display_name": "kubernetes-my-app-sa",
    "policies": ["default", "app-policy"],
    "metadata": {
      "role": "my-app-role",
      "service_account_name": "my-app-sa"
    }
  },
  "request": {
    "id": "req-uuid-123",
    "operation": "read",
    "path": "database/creds/my-role",
    "remote_address": "10.0.1.5"
  },
  "response": {
    "data": {
      "username": "hmac-sha256:ghi789...",  // Secret value HMAC'd
      "password": "hmac-sha256:jkl012..."   // Never logged in clear
    }
  }
}
```

### HMAC Hashing — Why and How

Secret values are hashed with HMAC-SHA256 using a per-cluster salt. This means:
- You can **correlate** audit events involving the same token or secret (same hash = same value)
- You cannot **recover** the original value from the hash
- Audit logs are safe to send to a SIEM without exposing credentials

```bash
# Verify a suspected token against audit logs
# Given a token s.XXXXX — hash it to match audit log entries
vault audit hash sys -format=json \
  input="s.XXXXX"
# Output: hmac-sha256:abc123...
# Compare against audit log's client_token field
```

### Compliance Use Cases

```bash
# Query audit logs for all accesses to production database credentials
grep '"path":"database/creds/prod' /var/log/vault/audit.log | \
  jq -r '.auth.display_name + " at " + .time'

# Example output:
# kubernetes-payment-service-sa at 2024-01-15T10:00:00Z
# terraform-ci-pipeline at 2024-01-15T10:01:00Z

# Alert on unusual patterns: same token from two different IPs
cat /var/log/vault/audit.log | \
  jq -r 'select(.type=="request") | [.auth.accessor, .request.remote_address] | @csv' | \
  sort | uniq | awk -F',' '{ count[$1]++ } END { for (a in count) if (count[a]>1) print a " - MULTIPLE IPs" }'
```

---

<a name="q28"></a>
## Q28 — Vault + Terraform at Scale: Secret Injection Patterns for Multi-Team, Multi-Env Platforms

**The question:** *"You're building an internal developer platform where 20+ teams each deploy services to dev/staging/prod. How do you structure Vault and Terraform so teams are isolated but the platform is manageable?"*

### Platform Architecture

```
Vault Namespace Structure:
  root/
    └── platform/          ← platform team manages this
          ├── shared/      ← shared secrets (shared DB, message broker)
          └── tenants/
                ├── team-payments/
                ├── team-orders/
                └── team-identity/

Each tenant namespace has:
  - Its own auth methods (Kubernetes auth bound to team's namespace)
  - Its own KV and database secret engines
  - Its own policies (team-scoped)
  - Cannot see other teams' namespaces
```

### Terraform Module for Provisioning a Tenant

```hcl
# modules/vault-tenant/main.tf
# Platform team uses this module to onboard each new team

resource "vault_namespace" "tenant" {
  path = "tenants/${var.team_name}"
}

# Kubernetes auth for this team's namespace
resource "vault_auth_backend" "kubernetes" {
  namespace = vault_namespace.tenant.path
  type      = "kubernetes"
}

resource "vault_kubernetes_auth_backend_config" "tenant" {
  namespace   = vault_namespace.tenant.path
  backend     = vault_auth_backend.kubernetes.path
  kubernetes_host = var.kubernetes_host
  # Service account tokens from the team's K8s namespace are auto-validated
}

# KV secrets engine for the team
resource "vault_mount" "kv" {
  namespace   = vault_namespace.tenant.path
  path        = "secret"
  type        = "kv-v2"
  description = "KV secrets for ${var.team_name}"
}

# Database credentials for the team's service DB
resource "vault_database_secret_backend_connection" "team_db" {
  namespace = vault_namespace.tenant.path
  backend   = "database"
  name      = "${var.team_name}-${var.environment}-postgres"

  postgresql {
    connection_url = "postgresql://{{username}}:{{password}}@${var.db_host}:5432/${var.team_name}"
    username       = var.db_admin_username
    password       = var.db_admin_password
  }
}

# Policy: team can only access their own secrets
resource "vault_policy" "team_app" {
  namespace = vault_namespace.tenant.path
  name      = "app-policy"
  policy    = <<-EOT
    path "secret/data/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    path "database/creds/*" {
      capabilities = ["read"]
    }
    path "auth/token/renew-self" {
      capabilities = ["update"]
    }
  EOT
}

# Kubernetes role: pods in the team's K8s namespace get app-policy
resource "vault_kubernetes_auth_backend_role" "app" {
  namespace                        = vault_namespace.tenant.path
  backend                          = vault_auth_backend.kubernetes.path
  role_name                        = "app-role"
  bound_service_account_names      = ["*"]
  bound_service_account_namespaces = [var.team_k8s_namespace]
  token_policies                   = ["app-policy"]
  token_ttl                        = 3600
}
```

### Onboarding a New Team

```hcl
# platform/tenants.tf — platform team manages one file
module "team_payments" {
  source = "./modules/vault-tenant"

  team_name          = "payments"
  environment        = "prod"
  kubernetes_host    = "https://k8s.prod.internal"
  team_k8s_namespace = "payments"
  db_host            = module.rds_payments.endpoint
  db_admin_username  = data.vault_kv_secret_v2.payments_db_admin.data["username"]
  db_admin_password  = data.vault_kv_secret_v2.payments_db_admin.data["password"]
}

module "team_orders" {
  source = "./modules/vault-tenant"

  team_name          = "orders"
  environment        = "prod"
  # ...
}
```

### Secret Rotation at Platform Scale

```bash
# Rotate ALL database credentials across all tenants simultaneously
# Vault handles this via role-based rotation

# Per-tenant DB rotation (safe — Vault creates new user, deletes old one atomically)
vault write -namespace=tenants/payments \
  database/rotate-root/payments-prod-postgres

# Audit who accessed what across all tenant namespaces
for ns in $(vault namespace list -namespace=tenants/ -format=json | jq -r '.[]'); do
  echo "=== Namespace: $ns ==="
  vault audit list -namespace="tenants/${ns}"
done
```

---

## 📋 Quick Reference Table — Key Terraform Concepts

| Concept | One-liner | Key gotcha |
|---------|-----------|------------|
| `for_each` over `count` | Stable string keys prevent index shift | Use `toset()` for list → set conversion |
| `create_before_destroy` | Create replacement before destroying old | Requires `name_prefix`, not fixed `name` |
| `moved` block | Rename resource in state without destroy | Keep moved blocks until all states migrated |
| `terraform_remote_state` | Read another layer's state outputs | Exposes entire state — prefer native data sources |
| `sensitive = true` | Hides value in terminal output only | State file still has plaintext — encrypt with KMS |
| Import block (1.5+) | Bring existing resources under management | Use `-generate-config-out` to auto-write HCL |
| `terraform_data` | Replacement for `null_resource` | Built-in — no provider needed |
| Workspaces | Separate state, same config | Not for prod isolation — use separate dirs |
| Provider alias | Multi-region/multi-account | Must explicitly `pass providers = {}` to modules |

## 📋 Quick Reference Table — Key Vault Concepts

| Concept | One-liner | Key gotcha |
|---------|-----------|------------|
| Batch tokens | Lightweight, no storage | Not renewable — use for short-lived, high-volume |
| Dynamic secrets | TTL-based credentials generated on demand | Every `terraform apply` generates a new cred — store in app, not Terraform |
| Response wrapping | One-time token — safe bootstrap delivery | Failed unwrap = interception alert |
| Transit engine | Encryption-as-a-service, Vault never stores data | `rewrap` existing ciphertexts after key rotation |
| Raft storage | Built-in HA, no Consul needed | Minimum 3 nodes for quorum |
| Namespaces | Multi-tenancy within one cluster | Enterprise feature — not open source |
| ACL `deny` | Overrides all other policies | Use to explicitly block cross-environment access |
| Audit fail-closed | Request blocked if audit device fails | Ensure audit storage never fills up |
| Entity + aliases | Unified identity across auth methods | External groups sync from LDAP on every login |
| PKI engine | Auto-rotating short-lived certs | Use intermediate CA not root CA for daily issuance |

---

> **Overall Interview Takeaway:** For Terraform, the depth markers are: `for_each` over `count` (with the index-shift explanation), lifecycle blocks (`create_before_destroy` naming gotcha), `moved` blocks for safe refactoring, import blocks for brownfield adoption, and the architectural choice between workspaces and separate state files. For Vault, the depth markers are: the token hierarchy and revocation cascade, dynamic secrets vs KV (TTL-based expiry), the Transit engine (encryption-as-a-service with key rotation), Vault Agent for K8s sidecar injection, and response wrapping for secure bootstrap. Combine both with the multi-team platform pattern (namespaces + Terraform modules for onboarding) to show senior-level design thinking.
