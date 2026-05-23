# HashiCorp Interview Question 1 — Secure Secrets Management with Terraform & Vault

> **Topic:** How to securely manage database credentials and secrets when using Terraform with HashiCorp Vault
> **Level:** Intermediate
> **Relevance:** A must-know pattern for any DevOps/Platform Engineer working with Terraform — interviewers will probe whether you hardcode secrets or manage them properly

---

## ❓ The Question

> **"You're creating an AWS RDS database with Terraform. Where do you store the database username and password? How do you make sure secrets never end up in your code or Git repository?"**

Follow-ups you'll commonly get:
- *"How does Terraform authenticate to Vault?"*
- *"What's the difference between static and dynamic database credentials in Vault?"*
- *"What alternatives exist if Vault isn't available?"*
- *"How do you prevent Terraform state from exposing secrets?"*

---

## 🧠 The Problem — Why Hardcoding Is Dangerous

The naive approach:

```hcl
# ❌ NEVER DO THIS
resource "aws_db_instance" "default" {
  username = "admin"
  password = "MyS3cr3tP@ssword!"   # Committed to Git → exposed forever
}
```

Even if you delete it later, Git history retains it. Anyone with repo access — or anyone who finds the repo via a breach — has your production database credentials. Credential leaks in Git are one of the top causes of cloud breaches.

Even using a `terraform.tfvars` file is risky:

```hcl
# terraform.tfvars — still risky
db_password = "MyS3cr3tP@ssword!"
# People forget to add it to .gitignore
# CI systems log variable values
# Terraform state stores it in plaintext
```

The correct answer: **secrets never touch your code or your pipeline environment variables. They come from a dedicated secrets manager at runtime.**

---

## 🖼️ Reference Diagram (Source: KodeKloud)

![Terraform and Vault integration — Vault provider, data block, secrets at runtime, not in GitHub](https://kodekloud.com/kk-media/image/upload/v1752873356/notes-assets/images/DevOps-Interview-Preparation-Course-Hashicorp-Question-1/terraform-vault-secrets-integration.jpg)

---

## 📖 Key Terminologies

| Term | Definition |
|------|-----------|
| **HashiCorp Vault** | Secrets management tool — stores, manages, and dynamically generates credentials |
| **Secret Engine** | Vault plugin that manages a specific type of secret (KV, database, AWS, PKI) |
| **KV Engine** | Key-Value secret engine — static secrets like API keys, passwords |
| **Dynamic Secrets** | Credentials generated on-demand by Vault (e.g., DB user with 1-hour TTL) |
| **Vault Provider** | Terraform plugin that enables Terraform to read from Vault |
| **Data Source** | Terraform block that reads existing data (not create it) — used to fetch secrets |
| **Lease** | Vault's TTL for a dynamic secret — Vault revokes it automatically when expired |
| **Auth Method** | How clients authenticate to Vault (AppRole, AWS IAM, Kubernetes, token) |
| **Vault Agent** | Sidecar/daemon that handles Vault authentication and secret rendering automatically |
| **AWS Secrets Manager** | AWS-native secrets store — alternative to Vault for AWS-only environments |

---

## ⚙️ The Solution — Terraform + Vault Integration

### Step 1: Store the Secret in Vault

```bash
# Enable the KV secrets engine (version 2)
vault secrets enable -path=secret kv-v2

# Store the database credentials
vault kv put secret/db \
  username="dbadmin" \
  password="$(openssl rand -base64 32)"

# Verify
vault kv get secret/db
```

### Step 2: Configure Vault Policy (Least Privilege)

Before Terraform can read from Vault, it needs a policy granting read access — nothing more:

```hcl
# vault-policy-terraform.hcl
path "secret/data/db" {
  capabilities = ["read"]
}

# Apply the policy
# vault policy write terraform-db-policy vault-policy-terraform.hcl
```

### Step 3: Terraform Configuration — The Right Way

```hcl
# providers.tf
terraform {
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.20"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "vault" {
  address = "https://vault.example.com"
  # Authentication: Vault token via environment variable VAULT_TOKEN
  # Or: use AppRole (preferred for CI/CD — see below)
}

provider "aws" {
  region = "us-east-1"
}

# Fetch credentials from Vault at runtime (not hardcoded)
data "vault_kv_secret_v2" "db_credentials" {
  mount = "secret"
  name  = "db"
}

resource "aws_db_instance" "default" {
  allocated_storage    = 20
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  db_name              = "myapp"

  # Credentials fetched dynamically from Vault — never in code
  username = data.vault_kv_secret_v2.db_credentials.data["username"]
  password = data.vault_kv_secret_v2.db_credentials.data["password"]

  parameter_group_name = "default.mysql8.0"
  skip_final_snapshot  = false

  tags = {
    Environment = "production"
  }
}
```

### How Terraform Authenticates to Vault

There are several methods — each suited for different environments:

**Method 1: Environment Variable (simplest, for local dev)**
```bash
export VAULT_ADDR="https://vault.example.com"
export VAULT_TOKEN="s.xxxxxxxxxxxxxxxxxxxxxxxx"
terraform apply
```

**Method 2: AppRole (recommended for CI/CD)**

AppRole is designed for machine-to-machine authentication. The pipeline has a Role ID (non-secret, like a username) and a Secret ID (secret, like a password):

```bash
# Create an AppRole
vault auth enable approle

vault write auth/approle/role/terraform-role \
  secret_id_ttl=10m \         # Secret ID expires after 10 minutes
  token_num_uses=10 \         # Token can be used 10 times
  token_ttl=20m \
  token_max_ttl=30m \
  policies="terraform-db-policy"

# Get the Role ID (store this in CI as a non-sensitive variable)
vault read auth/approle/role/terraform-role/role-id

# Generate a Secret ID (fetch fresh for each pipeline run)
vault write -f auth/approle/role/terraform-role/secret-id
```

```hcl
# Terraform AppRole authentication
provider "vault" {
  address = "https://vault.example.com"
  auth_login {
    path = "auth/approle/login"
    parameters = {
      role_id   = var.vault_role_id    # From CI env variable
      secret_id = var.vault_secret_id  # Fetched fresh per pipeline run
    }
  }
}
```

**Method 3: AWS IAM Auth (best for EC2/ECS/Lambda on AWS)**

```hcl
provider "vault" {
  address = "https://vault.example.com"
  auth_login_aws {
    role = "terraform-role"
    # Vault verifies the caller is the expected AWS IAM role
    # No credentials to manage — uses instance metadata
  }
}
```

**Method 4: Kubernetes Auth (for pipelines running in K8s)**

```hcl
provider "vault" {
  address = "https://vault.example.com"
  auth_login_kubernetes {
    role = "terraform-role"
    jwt  = file("/var/run/secrets/kubernetes.io/serviceaccount/token")
  }
}
```

---

## 🚀 Advanced: Dynamic Database Credentials

Static credentials (stored in KV) are good. **Dynamic credentials** are better. Vault can connect directly to your RDS instance and create a **temporary database user** with a short TTL — no permanent credentials exist at all.

```bash
# Enable database secrets engine
vault secrets enable database

# Configure Vault to connect to RDS MySQL
vault write database/config/my-rds-mysql \
  plugin_name=mysql-rds-database-plugin \
  connection_url="{{username}}:{{password}}@tcp(mydb.xxx.us-east-1.rds.amazonaws.com:3306)/" \
  allowed_roles="terraform-role" \
  username="vault-admin" \
  password="vault-admin-password"

# Define a role with a SQL template for user creation
vault write database/roles/terraform-role \
  db_name=my-rds-mysql \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';
                       GRANT SELECT, INSERT, UPDATE ON myapp.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```

```hcl
# Terraform reads a dynamic credential — valid for 1 hour
data "vault_database_secret_backend_creds" "db" {
  backend = "database"
  role    = "terraform-role"
}

resource "aws_db_instance" "default" {
  username = data.vault_database_secret_backend_creds.db.username
  password = data.vault_database_secret_backend_creds.db.password
  # These credentials expire after 1 hour — no permanent password exists
}
```

**Why dynamic credentials are superior:**
- No permanent credentials to rotate, leak, or forget about
- Every access is audited in Vault — you know exactly who got which credential and when
- If a pipeline is compromised, the credential it received expired hours ago

---

## 🔐 The Terraform State Problem

Even when you fetch secrets from Vault correctly, there's one more trap: **Terraform stores everything in state**.

```json
// terraform.tfstate — contains secrets in plaintext!
{
  "resources": [{
    "type": "aws_db_instance",
    "instances": [{
      "attributes": {
        "username": "dbadmin",
        "password": "MyS3cr3tP@ssword!"   ← stored in state!
      }
    }]
  }]
}
```

**How to protect state:**

```hcl
# Store state in S3 with encryption + DynamoDB lock
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/rds/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true          # AES-256 encryption at rest
    kms_key_id     = "arn:aws:kms:us-east-1:123:key/abc"  # KMS key
    dynamodb_table = "terraform-lock"
  }
}
```

Additional measures:
- Set strict S3 bucket policy — only the CI role and specific engineers can read state
- Enable S3 versioning — recover from accidental state corruption
- Use `sensitive = true` on all secret variables:

```hcl
variable "db_password" {
  type      = string
  sensitive = true   # Terraform will not print this in plan/apply output
}
```

---

## 🔄 Alternatives if Vault Isn't Available

| Option | How It Works | Pros | Cons |
|--------|-------------|------|------|
| **AWS Secrets Manager** | Managed AWS secrets store, native rotation | No infra, native AWS integration, auto-rotation | AWS-only, cost per secret |
| **AWS Parameter Store (SecureString)** | SSM parameter with KMS encryption | Free for standard, KMS encrypted | Less feature-rich than Secrets Manager |
| **Terraform Cloud/Enterprise** | Built-in variable encryption and workspace isolation | Integrated with Terraform workflow | Paid for large teams |
| **SOPS + KMS** | Encrypt secrets files in Git with AWS KMS key | Secrets live in Git (encrypted) | Key management overhead |
| **HashiCorp Vault** | Self-hosted or HCP Vault, full features | Dynamic secrets, fine-grained audit, multi-cloud | More setup, self-managed overhead |

**Never use:** S3 plaintext files, `.tfvars` in Git, environment variables logged by CI, hardcoded values.

### AWS Secrets Manager Alternative

```hcl
# Using AWS Secrets Manager instead of Vault
data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = "prod/myapp/db-credentials"
}

locals {
  db_creds = jsondecode(
    data.aws_secretsmanager_secret_version.db_credentials.secret_string
  )
}

resource "aws_db_instance" "default" {
  username = local.db_creds["username"]
  password = local.db_creds["password"]
}
```

---

## ⚠️ Common Gotchas

| Gotcha | Problem | Fix |
|--------|---------|-----|
| **VAULT_TOKEN in CI logs** | CI system prints environment variables → token leaked | Use AppRole; never pass tokens as `echo $VAULT_TOKEN` |
| **State stores the password** | Even with Vault, the RDS resource writes password to state | Encrypt state with KMS; restrict state access via IAM |
| **Vault token expires mid-apply** | Long `terraform apply` runs can outlast the Vault token | Use long-TTL tokens for CI or renew token before apply |
| **KV v1 vs KV v2 data path** | v1: `secret/db`, v2: `secret/data/db` — wrong path causes silent empty data | Check version: `vault secrets list -detailed` |
| **Dynamic creds cause drift** | Each `terraform apply` generates a new DB user if using dynamic creds | Dynamic creds are best for app runtime, not infra provisioning; use KV for RDS creation |
| **`sensitive` doesn't protect state** | `sensitive = true` hides output from terminal but state still stores plaintext | Always encrypt state separately |

---

## ✅ Best Practices

- **Never hardcode secrets** — not in `.tf` files, not in `terraform.tfvars`, not in CI environment variables that get logged
- **Use Vault AppRole** for CI/CD pipelines — generate a fresh Secret ID per pipeline run; it expires in minutes
- **Encrypt Terraform state** with S3 + KMS; restrict bucket access to the minimum required IAM roles
- **Mark all secret variables as `sensitive = true`** — prevents accidental log output
- **Prefer dynamic secrets over static** where possible — they expire automatically and are fully audited
- **Audit Vault access logs** — Vault logs every read; use these for compliance and incident response
- **Use Vault namespaces** to isolate dev/staging/prod secret paths — prevent a dev pipeline from reading prod secrets

---

## 📋 Quick Reference

```bash
# Store a secret in Vault KV v2
vault kv put secret/db username="admin" password="$(openssl rand -base64 32)"

# Read it back
vault kv get secret/db
vault kv get -field=password secret/db

# Create AppRole for Terraform CI
vault write auth/approle/role/terraform-role \
  policies="terraform-db-policy" secret_id_ttl=10m token_ttl=20m

vault read -field=role_id auth/approle/role/terraform-role/role-id
vault write -field=secret_id -f auth/approle/role/terraform-role/secret-id

# Run Terraform with Vault AppRole
export VAULT_ADDR="https://vault.example.com"
export TF_VAR_vault_role_id="<role-id>"
export TF_VAR_vault_secret_id="<secret-id>"
terraform apply

# Check what Terraform fetched from Vault (without showing the value)
terraform state show data.vault_kv_secret_v2.db_credentials
```

---

> **Interview Takeaway:** The answer to "where do you store secrets in Terraform" is never "in the code" or "in tfvars." Always say: secrets come from a dedicated secrets manager (Vault, AWS Secrets Manager) via a `data` source at runtime. Then go further: mention the Terraform state problem (state stores secrets in plaintext → encrypt with KMS), AppRole for CI authentication (not raw tokens), and dynamic credentials if you want to impress. These three layers — secret fetch, state encryption, short-lived auth — are what a complete answer looks like.
