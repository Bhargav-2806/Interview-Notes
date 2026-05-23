# Terraform Interview Question 2 — Variable Validation & Type Constraints

> **Topic:** Enforcing variable validation in Terraform during the plan phase — catching errors before `apply`
> **Level:** Intermediate
> **Relevance:** Shows you write defensive, production-grade Terraform — not just "it works on my machine" configs

---

## ❓ The Question

> **"How do you validate Terraform variables at plan time? Show an example for an AMI ID. What other kinds of validation have you implemented?"**

Follow-ups you'll commonly hear:
- *"What's the difference between type constraints and validation blocks?"*
- *"What Terraform functions are useful inside validation conditions?"*
- *"How does variable validation relate to Sentinel policies?"*
- *"How do you validate complex objects like a list of CIDR blocks?"*

---

## 🧠 Why Validate Variables at Plan Time?

Without validation, a bad variable value fails during `terraform apply` — after infrastructure might already be partially created. Worse, the error message from AWS/Azure is cryptic:

```
Error: InvalidAMIID.Malformed: Invalid id "12345" (expecting "ami-...")
```

With a validation block, Terraform catches it at `terraform plan` before anything is created:

```
Error: Invalid value for variable
  on main.tf line 3, in variable "ami":
   3: variable "ami" {
    ├────────────────
    │ var.ami is "12345"

  AMI ID must start with 'ami-' followed by 8 or 17 hex characters.
  Example: ami-0c02fb55956c7d316
```

This is the difference between a 2-second plan failure and a 5-minute apply failure that leaves orphaned resources.

---

## 🖼️ Reference Diagram (Source: KodeKloud)

![Terraform variable validation — AMI variable with validation block, plan-time error catching](https://kodekloud.com/kk-media/image/upload/v1752873358/notes-assets/images/DevOps-Interview-Preparation-Course-Terraform-Question-2/terraform-validate-ami-variable-notes.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **`validation` block** | A block inside a `variable` that defines conditions the value must satisfy |
| **`condition`** | A boolean expression that must be `true` for the variable to be accepted |
| **`error_message`** | Custom message shown to the user when validation fails |
| **Type constraint** | Terraform's built-in type system — `string`, `number`, `bool`, `list`, `map`, `object` |
| **`can()` function** | Returns true if an expression succeeds without error — useful for testing `regex()` matches |
| **`regex()` function** | Applies a regular expression — errors if no match (use with `can()` in validation) |
| **`contains()` function** | Returns true if a list contains a given value — for allowed-value checks |
| **`Sentinel`** | HashiCorp's policy-as-code framework (Terraform Cloud/Enterprise) for org-wide policy enforcement |

---

## ⚙️ The Validation Block — Syntax and Rules

```hcl
variable "variable_name" {
  type        = string
  description = "What this variable represents"
  default     = "optional-default-value"

  validation {
    condition     = <boolean expression using var.variable_name>
    error_message = "Must be at least one sentence. Must end with a period."
  }

  # Multiple validation blocks are allowed
  validation {
    condition     = <second condition>
    error_message = "Another requirement."
  }
}
```

**Rules:**
- `condition` must evaluate to `true` or `false` — any other result is an error
- `error_message` must be a non-empty string ending with a period or question mark
- The expression can only reference `var.<this_variable>` — not other variables or resources
- Multiple `validation` blocks are evaluated independently — all must pass

---

## 🛠️ Practical Validation Examples

### AMI ID (The Classic Example)

```hcl
variable "ami" {
  type        = string
  description = "EC2 AMI ID"
  default     = "ami-0c02fb55956c7d316"

  validation {
    # AMI IDs are "ami-" followed by exactly 8 or 17 hex characters
    condition = can(regex("^ami-[0-9a-f]{8}([0-9a-f]{9})?$", var.ami))
    error_message = "AMI ID must start with 'ami-' followed by 8 or 17 hex characters. Example: ami-0c02fb55956c7d316."
  }
}
```

The KodeKloud example uses `substr()` which only checks the prefix. The `regex` + `can()` approach is stricter — it validates the full format.

### Environment Name (Allowed Values)

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production."
  }
}
```

### Instance Type (Allowed List)

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.medium"

  validation {
    condition = contains([
      "t3.micro", "t3.small", "t3.medium", "t3.large",
      "m5.large", "m5.xlarge", "m5.2xlarge",
      "c5.large", "c5.xlarge",
    ], var.instance_type)
    error_message = "Instance type must be from the approved list. Contact the platform team to add new types."
  }
}
```

### CIDR Block Format

```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block, e.g., 10.0.0.0/16."
  }

  validation {
    # Ensure the prefix is between /16 and /24 (org policy)
    condition     = tonumber(split("/", var.vpc_cidr)[1]) >= 16 && tonumber(split("/", var.vpc_cidr)[1]) <= 24
    error_message = "VPC CIDR prefix must be between /16 and /24 per network policy."
  }
}
```

### AWS Region

```hcl
variable "aws_region" {
  type        = string
  description = "AWS region for deployment"

  validation {
    condition = can(regex("^(us|eu|ap|sa|ca|me|af)-(north|south|east|west|central|northeast|southeast)-[1-9]$", var.aws_region))
    error_message = "Must be a valid AWS region name, e.g., us-east-1, eu-west-2."
  }
}
```

### Port Number

```hcl
variable "app_port" {
  type        = number
  description = "Application listening port"
  default     = 8080

  validation {
    condition     = var.app_port >= 1024 && var.app_port <= 65535
    error_message = "Port must be between 1024 and 65535. Ports below 1024 require root privileges."
  }
}
```

### S3 Bucket Name (AWS Naming Rules)

```hcl
variable "bucket_name" {
  type        = string
  description = "S3 bucket name"

  validation {
    # S3 rules: 3-63 chars, lowercase letters/numbers/hyphens, no consecutive hyphens, no leading/trailing hyphens
    condition = can(regex("^[a-z0-9][a-z0-9-]{1,61}[a-z0-9]$", var.bucket_name)) && !can(regex("--", var.bucket_name))
    error_message = "S3 bucket name must be 3-63 characters, lowercase alphanumeric and hyphens only, no consecutive hyphens, no leading/trailing hyphens."
  }
}
```

### Non-Empty String (Prevent Accidental Empty Values)

```hcl
variable "app_name" {
  type        = string
  description = "Application name used in resource naming"

  validation {
    condition     = length(trimspace(var.app_name)) > 0
    error_message = "Application name must not be empty or whitespace-only."
  }

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,30}[a-z0-9]$", var.app_name))
    error_message = "Application name must be lowercase, start with a letter, contain only letters/numbers/hyphens, and be 3-32 characters."
  }
}
```

---

## 🔢 Type Constraints — Beyond Basic `string`

Type constraints catch type errors before validation even runs:

```hcl
# Simple types
variable "instance_count" {
  type    = number
  default = 2
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

# List types
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones required for high availability."
  }
}

# Map type
variable "tags" {
  type    = map(string)
  default = {}

  validation {
    condition     = contains(keys(var.tags), "Environment")
    error_message = "Tags map must include an 'Environment' key."
  }
}

# Object type — structured input with specific field types
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })

  default = {
    engine         = "mysql"
    engine_version = "8.0"
    instance_class = "db.t3.medium"
    storage_gb     = 100
    multi_az       = true
  }

  validation {
    condition     = contains(["mysql", "postgres", "mariadb"], var.database_config.engine)
    error_message = "Database engine must be mysql, postgres, or mariadb."
  }

  validation {
    condition     = var.database_config.storage_gb >= 20 && var.database_config.storage_gb <= 65536
    error_message = "Storage must be between 20 and 65536 GB."
  }
}

# Optional fields in objects (Terraform 1.3+)
variable "backup_config" {
  type = object({
    enabled           = bool
    retention_days    = optional(number, 7)    # default 7 if not provided
    backup_window     = optional(string, "03:00-04:00")
  })
}
```

---

## 🔐 Sensitive Variables

```hcl
variable "db_password" {
  type        = string
  description = "RDS master password"
  sensitive   = true    # Terraform will NOT print this in plan/apply output

  validation {
    condition     = length(var.db_password) >= 16
    error_message = "Database password must be at least 16 characters."
  }

  validation {
    # Must contain uppercase, lowercase, number, and special char
    condition = can(regex("[A-Z]", var.db_password)) && can(regex("[a-z]", var.db_password)) && can(regex("[0-9]", var.db_password)) && can(regex("[^a-zA-Z0-9]", var.db_password))
    error_message = "Password must contain uppercase, lowercase, number, and special character."
  }
}
```

`sensitive = true` prevents the value from appearing in:
- `terraform plan` output
- `terraform apply` output
- State file display via `terraform show`

> Note: The value IS still stored in the state file. Always encrypt state (S3 + KMS) for sensitive variables.

---

## 🆚 `can()` vs Direct Expression

Many validation conditions involve functions that throw errors on bad input rather than returning false. `can()` wraps these:

```hcl
# Without can() — would throw an error (not return false) on bad input
condition = regex("^ami-", var.ami) != null   # ❌ regex() errors if no match

# With can() — returns false gracefully when regex finds no match
condition = can(regex("^ami-[0-9a-f]{8,17}$", var.ami))   # ✅ returns false on no match
```

Functions that need `can()` wrapping in validations:
- `regex()` — errors on no match
- `cidrnetmask()` — errors on invalid CIDR
- `tonumber()` — errors on non-numeric string
- `toset()` — errors on incompatible types
- `jsondecode()` — errors on invalid JSON

---

## 📊 Variable Validation vs Sentinel Policies

Variable validation and Sentinel solve overlapping but different problems:

| Aspect | Variable Validation | Sentinel (Terraform Cloud/Enterprise) |
|--------|--------------------|-----------------------------------------|
| **Where it runs** | In the Terraform config | On Terraform Cloud/Enterprise |
| **Enforced by** | Module author | Organization/platform team |
| **Scope** | Single variable in a module | Entire plan — any resource, attribute |
| **Can check** | Variable format/value | Any plan attribute (e.g., all S3 buckets must be encrypted) |
| **Bypass** | Module user can override | Hard enforcement — cannot bypass |
| **Cost** | Free (OSS Terraform) | Requires Terraform Cloud/Enterprise |
| **Example** | "AMI must start with ami-" | "No EC2 instance can use instance type larger than m5.xlarge without approval" |

Use variable validation for format checking within a module. Use Sentinel (or OPA with Conftest) for org-wide policy enforcement across all Terraform plans.

---

## 🌍 Real-World Usage — Module with Full Validation

In practice, a reusable Terraform module should validate all its inputs:

```hcl
# modules/ec2-instance/variables.tf

variable "name" {
  type        = string
  description = "Name for the EC2 instance (used in tags and naming)"

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,30}[a-z0-9]$", var.name))
    error_message = "Name must be lowercase alphanumeric with hyphens, 3-32 characters."
  }
}

variable "ami_id" {
  type        = string
  description = "AMI ID to use for the instance"

  validation {
    condition     = can(regex("^ami-[0-9a-f]{8}([0-9a-f]{9})?$", var.ami_id))
    error_message = "Must be a valid AMI ID (ami-xxxxxxxx or ami-xxxxxxxxxxxxxxxxx)."
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"

  validation {
    condition     = can(regex("^(t3|t3a|m5|m5a|c5|r5)\\.(micro|small|medium|large|xlarge|2xlarge)$", var.instance_type))
    error_message = "Instance type must be from the approved families: t3, t3a, m5, m5a, c5, r5."
  }
}

variable "subnet_id" {
  type        = string
  description = "Subnet ID to launch the instance in"

  validation {
    condition     = can(regex("^subnet-[0-9a-f]{8,17}$", var.subnet_id))
    error_message = "Must be a valid subnet ID (subnet-xxxxxxxx)."
  }
}

variable "root_volume_size_gb" {
  type        = number
  description = "Root EBS volume size in GB"
  default     = 30

  validation {
    condition     = var.root_volume_size_gb >= 20 && var.root_volume_size_gb <= 500
    error_message = "Root volume size must be between 20 and 500 GB."
  }
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply to all resources"
  default     = {}

  validation {
    condition     = contains(keys(var.tags), "Team") && contains(keys(var.tags), "CostCenter")
    error_message = "Tags must include 'Team' and 'CostCenter' keys for cost allocation."
  }
}
```

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **Validation references other variables** | `condition = var.ami != "" && var.region != ""` — not allowed | Each validation can only reference its own `var.<name>` |
| **`error_message` doesn't end with punctuation** | Terraform rejects the config | Always end error messages with a period or question mark |
| **Using `regex()` without `can()`** | Regex that doesn't match throws an error instead of returning false | Always wrap `regex()` in `can()` inside validation conditions |
| **Type constraint vs validation order** | If type is wrong (e.g., passing a number for a string variable), type check runs first — validation never runs | This is actually correct behavior — be aware of the order |
| **Default value bypasses validation** | A wrong default value that fails validation will error immediately | Ensure defaults also pass all validation conditions |
| **Overly strict regex** | Region regex doesn't account for `ap-southeast-3` → valid region rejected | Test regexes against the full range of valid values before committing |

---

## ✅ Best Practices

- **Validate all module inputs** — modules are shared; you can't control how callers use them
- **Use `regex()` + `can()` for format validation** — more precise than `substr()` + `length()`
- **Write descriptive error messages** — tell the user the correct format with an example (`Example: ami-0c02fb55956c7d316`)
- **Mark sensitive variables with `sensitive = true`** — prevents values from appearing in logs; encrypt state separately
- **Use `object()` types for related variables** — group database config, network config, etc. into a single structured variable instead of 10 separate strings
- **Use `optional()` in objects** (Terraform 1.3+) for fields with sensible defaults
- **Test validations with `terraform plan -var`** before finalising — validate both valid and invalid inputs

---

## 📋 Quick Reference — Common Validation Patterns

```hcl
# Format check with regex
condition = can(regex("^ami-[0-9a-f]{8,17}$", var.ami))

# Allowed values list
condition = contains(["dev", "staging", "prod"], var.env)

# Length range
condition = length(var.name) >= 3 && length(var.name) <= 32

# Number range
condition = var.port >= 1024 && var.port <= 65535

# Non-empty string
condition = length(trimspace(var.value)) > 0

# Valid CIDR block
condition = can(cidrnetmask(var.cidr))

# Required map key
condition = contains(keys(var.tags), "Environment")

# Starts with prefix (KodeKloud's original approach)
condition = length(var.ami) > 4 && substr(var.ami, 0, 4) == "ami-"

# String matches one of several prefixes
condition = startswith(var.bucket, "prod-") || startswith(var.bucket, "staging-")
```

---

> **Interview Takeaway:** Variable validation turns Terraform from "fails at apply with a cryptic cloud error" to "fails at plan with a clear, actionable message." Show the interviewer you use `can(regex(...))` for format validation, `contains()` for allowed-value lists, and `object()` types for structured inputs. Then mention: validation only checks the variable itself — for org-wide policies across all resources in a plan, that's where Sentinel or OPA/Conftest comes in. That distinction shows you understand the full spectrum.
