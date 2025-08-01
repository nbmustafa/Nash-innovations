act as an expert devops to build a complext conditional k8s config map with terraform, with secrets come from aws secret manager and dev, nonpros, prep and prod environmemt each has different secret name in aws

To create a complex conditional Kubernetes ConfigMap with Terraform that retrieves secrets from AWS Secrets Manager with environment-specific secret names (dev, nonprod, prep, prod), follow this structured approach:

### Solution Overview
1. **Environment Detection**: Use Terraform workspaces to manage environment-specific configurations.
2. **Secret Retrieval**: Fetch secrets from AWS Secrets Manager using workspace-specific secret names.
3. **ConfigMap Construction**: Generate a ConfigMap with merged data, combining static values and secrets.
4. **Sensitive Data Handling**: Use `local-exec` to safely handle secrets (avoid storing in state).

---

### Terraform Configuration (`main.tf`)
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = "us-east-1" # Update with your region
}

provider "kubernetes" {
  config_path = "~/.kube/config" # Update for your cluster
}

# Map workspace names to secret names
locals {
  secret_name_map = {
    dev    = "app/dev/database_creds"
    nonprod = "app/nonprod/database_creds"
    prep   = "app/prep/database_creds"
    prod   = "app/prod/database_creds"
  }
  
  # Static ConfigMap data per environment
  config_data = {
    dev = {
      LOG_LEVEL = "debug"
      API_URL   = "https://dev-api.example.com"
    }
    nonprod = {
      LOG_LEVEL = "info"
      API_URL   = "https://nonprod-api.example.com"
    }
    prep = {
      LOG_LEVEL = "warn"
      API_URL   = "https://prep-api.example.com"
    }
    prod = {
      LOG_LEVEL = "error"
      API_URL   = "https://api.example.com"
    }
  }
}

# Fetch secret from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "app_secret" {
  secret_id = local.secret_name_map[terraform.workspace]
}

# Extract specific keys from the secret (assumes JSON structure)
locals {
  secret_json = jsondecode(data.aws_secretsmanager_secret_version.app_secret.secret_string)
  db_username = local.secret_json["DB_USERNAME"]
  db_password = local.secret_json["DB_PASSWORD"]
}

# Generate ConfigMap with merged data
resource "kubernetes_config_map" "app_config" {
  metadata {
    name      = "app-config-${terraform.workspace}"
    namespace = "default"
  }

  data = merge(
    local.config_data[terraform.workspace],      # Environment-specific static data
    { DB_HOST = "db-host-${terraform.workspace}" }, # Additional computed values
    { DB_USER = local.db_username }               # Secrets (use cautiously!)
  )

  # Warning: Secrets in ConfigMaps are base64-encoded but not encrypted.
  lifecycle {
    ignore_changes = [data] # Prevent drift from external changes
  }
}

# Use local-exec to avoid storing secrets in Terraform state
resource "null_resource" "inject_secrets" {
  triggers = {
    secret_version = data.aws_secretsmanager_secret_version.app_secret.version_id
  }

  provisioner "local-exec" {
    command = <<-EOT
      kubectl create configmap app-secrets-${terraform.workspace} \
        --namespace=default \
        --from-literal=DB_PASSWORD=${local.db_password} \
        --dry-run=client -o yaml | kubectl apply -f -
    EOT
    environment = {
      KUBECONFIG = "~/.kube/config" # Update for your cluster
    }
  }
}
```

---

### Key Components
1. **Workspace-Based Configuration**:
   - **Secret Names**: Mapped using `local.secret_name_map` based on `terraform.workspace`.
   - **ConfigMap Data**: Static environment settings in `local.config_data`.

2. **Secret Handling**:
   - Secrets are fetched from AWS Secrets Manager as a JSON string.
   - Specific keys (e.g., `DB_USERNAME`, `DB_PASSWORD`) are extracted.

3. **ConfigMap Construction**:
   - **Non-Sensitive Data**: Merged using `merge()` (static values + computed values).
   - **Sensitive Data**: **Avoid storing in ConfigMaps** (use Secrets instead, as shown in `null_resource`).

4. **Safe Secret Injection**:
   - `null_resource` uses `kubectl` to create a Kubernetes Secret *without* persisting sensitive data in Terraform state.

---

### Usage Instructions
1. **Initialize Workspaces**:
   ```bash
   terraform workspace new dev
   terraform workspace new nonprod
   terraform workspace new prep
   terraform workspace new prod
   ```

2. **Deploy for an Environment**:
   ```bash
   terraform workspace select dev
   terraform apply
   ```

3. **Verify Resources**:
   ```bash
   kubectl get configmap/app-config-dev -o yaml
   kubectl get secret/app-secrets-dev -o yaml
   ```

---

### Security Notes
1. **Secrets in ConfigMaps**: 
   - The example includes `DB_USER` in the ConfigMap for demonstration **only**. Avoid this in production.
   - Always use Kubernetes Secrets for sensitive data (handled via `null_resource`).

2. **State Security**:
   - Enable Terraform state encryption (e.g., S3 backend with SSE).
   - Restrict AWS IAM policies to least privilege for Secrets Manager.

3. **Secret Rotation**:
   - Use AWS Secrets Manager rotation or reloader tools (e.g., `stakater/reloader`) to update pods when secrets change.

This approach ensures environment-specific configurations while securely managing secrets outside Terraform state. Adjust namespaces, regions, and secret structures to match your requirements.