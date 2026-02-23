### Terraform + IaaS for DBA — Complete Interview Guide
### AWS RDS • Aurora • GCP Cloud SQL • Terraform Best Practices • Production Architecture

> **75+ In-Depth Questions with Detailed Answers, Terraform Code & Architecture Diagrams**
> For Sr Cloud DBA / DevOps DBA / Database Infrastructure Engineer Interviews

---
---

### SECTION 1: CORE TERRAFORM CONCEPTS

---

### Q1. What is Terraform and why do DBAs need it?

**Answer:** Terraform is an Infrastructure as Code (IaC) tool by HashiCorp that lets you define, provision, and manage cloud infrastructure using declarative configuration files (HCL — HashiCorp Configuration Language).

**Why DBAs need Terraform:**
- **Consistency:** Same database infrastructure across DEV, QA, STAGING, PROD
- **Version Control:** Database infrastructure changes tracked in Git (who changed what, when)
- **Repeatability:** Spin up identical RDS/Aurora/Cloud SQL instances in minutes
- **Disaster Recovery:** Recreate entire database infrastructure from code after a disaster
- **Compliance:** Enforce encryption, private networking, backup policies via code
- **Cost Management:** Destroy non-production environments when not needed

```
  WITHOUT Terraform:                    WITH Terraform:
  ─────────────────                     ──────────────
  DBA clicks through AWS Console        DBA writes main.tf:
  → Creates RDS manually                  resource "aws_db_instance" "prod" {
  → Forgets to enable encryption              engine = "postgres"
  → Different settings in each env            multi_az = true
  → No record of what was done               storage_encrypted = true
  → Can't reproduce after disaster            ...
  → "It worked on my screen"              }
                                         → terraform apply
                                         → Identical in every environment
                                         → Full audit trail in Git
                                         → Recreate in 10 minutes after disaster
```

---

### Q2. Explain the Terraform lifecycle (init, plan, apply, destroy).

**Answer:**

```
  TERRAFORM LIFECYCLE:

  Step 1: terraform init
  ├── Downloads provider plugins (aws, google, azurerm)
  ├── Initializes backend (local or remote state)
  ├── Downloads modules from registry
  └── Creates .terraform/ directory

  Step 2: terraform plan
  ├── Reads current state (terraform.tfstate)
  ├── Compares desired config (.tf files) with actual state
  ├── Shows what will be CREATED, CHANGED, or DESTROYED
  ├── Does NOT make any changes (dry run)
  └── Output: "Plan: 3 to add, 1 to change, 0 to destroy"

  Step 3: terraform apply
  ├── Executes the planned changes
  ├── Creates/modifies/destroys resources in the cloud
  ├── Updates terraform.tfstate with new state
  └── Shows: "Apply complete! Resources: 3 added, 1 changed, 0 destroyed"

  Step 4: terraform destroy
  ├── Destroys ALL resources managed by this Terraform config
  ├── Updates state file (resources removed)
  └── Use with caution! (deletion_protection saves you in prod)
```

```bash
# Real-world workflow for a DBA:
cd /terraform/rds-production

terraform init              # First time: download AWS provider
terraform plan -out=plan.out  # Review what will change
# DBA reviews the plan output carefully...
terraform apply plan.out      # Execute the approved plan
terraform show               # Verify the created resources
```

---

### Q3. What is Terraform state file and why is it critical?

**Answer:** The state file (`terraform.tfstate`) is Terraform's **single source of truth**. It maps your Terraform configuration to real-world cloud resources.

```
  terraform.tfstate stores:
  ├── Resource IDs (i-0abc123, db-xyz789)
  ├── Current attribute values (instance_class, engine_version)
  ├── Dependencies between resources
  ├── Metadata (provider versions, serial number)
  └── Sensitive data (passwords — another reason to secure it!)

  WHY IT'S CRITICAL:
  ├── Without state, Terraform doesn't know what exists in the cloud
  ├── It would try to CREATE everything again (duplicate resources!)
  ├── State enables: plan (diff), import, drift detection
  └── Losing state = losing control of your infrastructure
```

### Local vs Remote State

```hcl
# LOCAL STATE (default — NOT recommended for teams)
# State file stored on your laptop: ./terraform.tfstate
# Problems: no sharing, no locking, risk of loss

# REMOTE STATE (recommended — S3 + DynamoDB)
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "rds/production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"   # prevents concurrent applies
  }
}

# GCP Remote State
terraform {
  backend "gcs" {
    bucket = "company-terraform-state"
    prefix = "cloudsql/production"
  }
}
```

### State Locking with DynamoDB

```hcl
# Create DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform State Lock Table"
  }
}
```

**Interview Question: "What happens if Terraform state file is corrupted?"**
- Pull the last good state from S3 versioning (enable versioning on the S3 bucket!)
- Use `terraform import` to re-import resources
- Use `terraform state rm` and `terraform import` to fix individual resources
- Last resort: delete state and re-import everything

---

### Q4. Explain Terraform modules and why they matter for database provisioning.

**Answer:** Modules are **reusable Terraform packages** — like functions in programming. You write the database config once and reuse it across environments.

```
  PROJECT STRUCTURE WITH MODULES:

  terraform-database-infra/
  ├── modules/
  │   ├── rds-postgres/           ← Reusable RDS PostgreSQL module
  │   │   ├── main.tf             ← RDS instance, subnet group, parameter group
  │   │   ├── variables.tf        ← Input variables (instance class, storage, etc.)
  │   │   ├── outputs.tf          ← Outputs (endpoint, port, ARN)
  │   │   └── security.tf         ← Security group rules
  │   │
  │   ├── aurora-cluster/         ← Reusable Aurora module
  │   │   ├── main.tf
  │   │   ├── variables.tf
  │   │   └── outputs.tf
  │   │
  │   └── monitoring/             ← CloudWatch alarms module
  │       ├── main.tf
  │       └── variables.tf
  │
  ├── environments/
  │   ├── dev/
  │   │   ├── main.tf             ← Calls module with dev settings
  │   │   └── terraform.tfvars    ← dev: db.t3.medium, 50GB, single-AZ
  │   ├── qa/
  │   │   ├── main.tf
  │   │   └── terraform.tfvars    ← qa: db.r6g.large, 100GB, multi-AZ
  │   └── prod/
  │       ├── main.tf
  │       └── terraform.tfvars    ← prod: db.r6g.2xlarge, 500GB, multi-AZ
  │
  └── global/
      └── backend.tf              ← Remote state configuration
```

### Module Usage Example

```hcl
# environments/prod/main.tf
module "rds_postgres" {
  source = "../../modules/rds-postgres"

  identifier          = "prod-telecom-pg"
  engine_version      = "16.3"
  instance_class      = "db.r6g.2xlarge"
  allocated_storage   = 500
  multi_az            = true
  storage_encrypted   = true
  backup_retention    = 35
  deletion_protection = true
  environment         = "production"

  vpc_id              = module.networking.vpc_id
  subnet_ids          = module.networking.private_subnet_ids
}

# environments/dev/main.tf — same module, different settings
module "rds_postgres" {
  source = "../../modules/rds-postgres"

  identifier          = "dev-telecom-pg"
  engine_version      = "16.3"
  instance_class      = "db.t3.medium"      # smaller for dev
  allocated_storage   = 50                   # less storage
  multi_az            = false                # no HA in dev
  storage_encrypted   = true
  backup_retention    = 1                    # minimal backups
  deletion_protection = false                # can destroy dev
  environment         = "development"

  vpc_id              = module.networking.vpc_id
  subnet_ids          = module.networking.private_subnet_ids
}
```

---

### Q5. Explain Terraform variables, outputs, and data sources.

```hcl
# ========================================
# variables.tf — Input Variables
# ========================================

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.r6g.large"

  validation {
    condition     = can(regex("^db\\.", var.db_instance_class))
    error_message = "Instance class must start with 'db.'"
  }
}

variable "db_password" {
  description = "Master database password"
  type        = string
  sensitive   = true   # won't show in plan output or logs
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "qa", "staging", "production"], var.environment)
    error_message = "Environment must be dev, qa, staging, or production."
  }
}

variable "backup_retention" {
  description = "Backup retention in days"
  type        = number
  default     = 7
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to connect"
  type        = list(string)
  default     = ["10.0.0.0/16"]
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {
    Team      = "DBA"
    ManagedBy = "Terraform"
  }
}

# ========================================
# outputs.tf — Output Values
# ========================================

output "rds_endpoint" {
  description = "RDS connection endpoint"
  value       = aws_db_instance.postgres.endpoint
}

output "rds_port" {
  description = "RDS port"
  value       = aws_db_instance.postgres.port
}

output "rds_arn" {
  description = "RDS ARN for IAM policies"
  value       = aws_db_instance.postgres.arn
  sensitive   = false
}

# ========================================
# data sources — Read existing resources
# ========================================

# Get existing VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Get password from Secrets Manager (never hardcode!)
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/rds/master-password"
}

# Get latest PostgreSQL engine version
data "aws_rds_engine_version" "postgres" {
  engine  = "postgres"
  version = "16"
  latest  = true
}

# Use in resource:
resource "aws_db_instance" "postgres" {
  engine_version = data.aws_rds_engine_version.postgres.version
  password       = data.aws_secretsmanager_secret_version.db_password.secret_string
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  # ...
}
```

---

### Q6. Explain depends_on, lifecycle rules, and prevent_destroy.

```hcl
# ========================================
# depends_on — Explicit dependency ordering
# ========================================
resource "aws_db_instance" "postgres" {
  # ...
  depends_on = [
    aws_db_subnet_group.main,      # subnet group must exist first
    aws_security_group.rds,        # security group must exist first
    aws_iam_role.rds_monitoring,   # monitoring role must exist first
  ]
}

# ========================================
# lifecycle — Control resource behavior
# ========================================
resource "aws_db_instance" "prod_postgres" {
  identifier          = "prod-postgres"
  deletion_protection = true        # AWS-level protection

  lifecycle {
    prevent_destroy = true          # Terraform-level: prevents 'terraform destroy'
                                    # terraform plan will ERROR if it tries to destroy this

    ignore_changes = [
      password,                     # Don't track password changes (rotated externally)
      latest_restorable_time,       # Changes constantly, ignore
    ]

    create_before_destroy = true    # For zero-downtime replacements:
                                    # Create new instance FIRST, then destroy old one
  }
}

# ========================================
# Preventing accidental deletion — 3 LAYERS
# ========================================
# Layer 1: AWS deletion_protection = true (API level)
# Layer 2: Terraform lifecycle { prevent_destroy = true } (IaC level)
# Layer 3: IAM policy denying rds:DeleteDBInstance (permission level)
```

---

### Q7. How do you import an existing RDS instance into Terraform?

**Answer:** When you have an RDS instance created manually (via console) and want to manage it with Terraform:

```bash
# Step 1: Write the Terraform configuration for the existing resource
# main.tf:
resource "aws_db_instance" "existing_postgres" {
  identifier     = "prod-postgres-legacy"
  engine         = "postgres"
  engine_version = "14.8"
  instance_class = "db.r6g.large"
  # ... match ALL current settings exactly
}

# Step 2: Import the resource into Terraform state
terraform import aws_db_instance.existing_postgres prod-postgres-legacy

# Step 3: Run plan to see if config matches reality
terraform plan
# If there are differences, update your .tf files to match the actual resource
# Goal: terraform plan shows "No changes. Infrastructure is up-to-date."

# Step 4: Now you can manage it with Terraform going forward!
```

**Interview tip:** After import, ALWAYS run `terraform plan` and fix any drift before making new changes.

---

### Q8. How do you handle drift detection?

**Answer:** Drift happens when someone manually changes a resource in the AWS Console that's managed by Terraform.

```bash
# Detect drift:
terraform plan
# Shows: "~ update in-place" for any drifted resources

# Example drift scenario:
# DBA manually changed instance_class from db.r6g.large to db.r6g.xlarge in AWS Console
# terraform plan output:
#   ~ resource "aws_db_instance" "postgres" {
#       ~ instance_class = "db.r6g.xlarge" -> "db.r6g.large"  # will revert!
#   }

# Options:
# 1. Accept the drift: Update your .tf file to match the new value
# 2. Revert the drift: terraform apply (will change it back to what's in .tf)
# 3. Refresh state: terraform refresh (updates state to match reality, but doesn't change .tf)
```

---
---

### SECTION 2: AWS RDS — DEEP DIVE

---

### Q9. What is the difference between Multi-AZ and Read Replica?

| Feature | Multi-AZ | Read Replica |
|---------|----------|--------------|
| **Purpose** | High Availability (HA) | Read scalability |
| **Replication** | Synchronous (RDS), Async (Aurora) | Asynchronous |
| **Readable** | No (standby is NOT accessible) | Yes (read-only queries) |
| **Failover** | Automatic (60-120 sec) | Manual promotion |
| **Endpoint** | Same endpoint (DNS swap on failover) | Separate endpoint per replica |
| **Region** | Same region, different AZ | Same region or cross-region |
| **Cost** | 2x primary cost | Per-replica cost |
| **Use Case** | Production HA / DR | Reporting, analytics, read-heavy apps |
| **Backup** | Taken from standby (no I/O impact) | Not used for backups |

```
  MULTI-AZ ARCHITECTURE:
  ┌─────────────┐    Synchronous    ┌─────────────┐
  │   PRIMARY   │ ──Replication──►  │   STANDBY    │
  │  (AZ-1a)    │                   │  (AZ-1b)     │
  │  Read/Write  │                  │  Not Readable│
  └──────┬───────┘                  └──────┬───────┘
         │                                 │
         │  ← Application connects here    │ ← Automatic failover
         │     (single DNS endpoint)       │    (DNS points here now)

  READ REPLICA ARCHITECTURE:
  ┌─────────────┐    Asynchronous   ┌─────────────┐
  │   PRIMARY    │ ──Replication──► │  REPLICA 1   │  ← Readable!
  │  (AZ-1a)     │                  │  (AZ-1b)     │
  │  Read/Write  │                  └──────────────┘
  └──────────────┘ ──Replication──► ┌──────────────┐
                                    │  REPLICA 2   │  ← Readable!
                                    │  (AZ-1c)     │
                                    └──────────────┘
```

---

### Q10. Complete Production RDS PostgreSQL with Terraform

```hcl
# ============================================================
# PRODUCTION RDS POSTGRESQL — COMPLETE TERRAFORM CONFIGURATION
# ============================================================

# --- Provider ---
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "telecom-terraform-state"
    key            = "rds/production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "ap-south-1"
  default_tags {
    tags = {
      Environment = "production"
      Team        = "DBA"
      ManagedBy   = "Terraform"
      Project     = "Telecom"
    }
  }
}

# --- Networking ---
resource "aws_db_subnet_group" "main" {
  name       = "prod-rds-subnet-group"
  subnet_ids = var.private_subnet_ids
  description = "Private subnets for RDS instances"

  tags = { Name = "prod-rds-subnet-group" }
}

resource "aws_security_group" "rds" {
  name_prefix = "prod-rds-sg-"
  vpc_id      = var.vpc_id
  description = "Security group for production RDS PostgreSQL"

  ingress {
    description     = "PostgreSQL from application subnets"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    cidr_blocks     = var.app_subnet_cidrs
  }

  ingress {
    description     = "PostgreSQL from bastion host"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.bastion_sg_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "prod-rds-postgresql-sg" }
}

# --- Parameter Group (PostgreSQL Tuning) ---
resource "aws_db_parameter_group" "postgres16" {
  name_prefix = "prod-postgres16-"
  family      = "postgres16"
  description = "Custom parameter group for production PostgreSQL 16"

  # Connection and Memory
  parameter { name = "max_connections"          value = "500"    apply_method = "pending-reboot" }
  parameter { name = "shared_buffers"           value = "{DBInstanceClassMemory/4}" apply_method = "pending-reboot" }
  parameter { name = "effective_cache_size"      value = "{DBInstanceClassMemory*3/4}" apply_method = "immediate" }
  parameter { name = "work_mem"                  value = "65536"  apply_method = "immediate" }
  parameter { name = "maintenance_work_mem"      value = "524288" apply_method = "immediate" }

  # WAL and Checkpoints
  parameter { name = "wal_buffers"              value = "16384"  apply_method = "pending-reboot" }
  parameter { name = "checkpoint_completion_target" value = "0.9" apply_method = "immediate" }
  parameter { name = "max_wal_size"             value = "4096"   apply_method = "immediate" }

  # Query Tuning
  parameter { name = "random_page_cost"         value = "1.1"    apply_method = "immediate" }
  parameter { name = "effective_io_concurrency"  value = "200"    apply_method = "immediate" }

  # Logging
  parameter { name = "log_min_duration_statement" value = "1000"  apply_method = "immediate" }
  parameter { name = "log_connections"            value = "1"      apply_method = "immediate" }
  parameter { name = "log_disconnections"         value = "1"      apply_method = "immediate" }
  parameter { name = "log_lock_waits"             value = "1"      apply_method = "immediate" }
  parameter { name = "log_statement"              value = "ddl"    apply_method = "immediate" }

  # Replication (for read replicas)
  parameter { name = "rds.logical_replication"   value = "1"      apply_method = "pending-reboot" }

  lifecycle { create_before_destroy = true }
}

# --- KMS Key for Encryption ---
resource "aws_kms_key" "rds" {
  description             = "KMS key for RDS encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  tags = { Name = "prod-rds-kms-key" }
}

resource "aws_kms_alias" "rds" {
  name          = "alias/prod-rds-encryption"
  target_key_id = aws_kms_key.rds.key_id
}

# --- Secrets Manager for Password ---
resource "aws_secretsmanager_secret" "rds_password" {
  name        = "prod/rds/postgres/master-password"
  description = "RDS PostgreSQL master password"
  kms_key_id  = aws_kms_key.rds.arn
}

resource "aws_secretsmanager_secret_version" "rds_password" {
  secret_id     = aws_secretsmanager_secret.rds_password.id
  secret_string = jsonencode({
    username = "pgadmin"
    password = var.db_password
    engine   = "postgres"
    host     = aws_db_instance.postgres.address
    port     = 5432
    dbname   = "telecom_db"
  })
}

# --- IAM Role for Enhanced Monitoring ---
resource "aws_iam_role" "rds_monitoring" {
  name = "prod-rds-monitoring-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "monitoring.rds.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  role       = aws_iam_role.rds_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}

# --- RDS INSTANCE ---
resource "aws_db_instance" "postgres" {
  identifier = "prod-telecom-postgres"

  # Engine
  engine         = "postgres"
  engine_version = "16.3"

  # Compute
  instance_class = "db.r6g.2xlarge"    # 8 vCPU, 64 GB RAM

  # Storage
  allocated_storage     = 500           # GB
  max_allocated_storage = 1000          # Auto-scaling up to 1TB
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  iops                  = 12000         # gp3 provisioned IOPS
  storage_throughput    = 500           # gp3 throughput (MB/s)

  # High Availability
  multi_az = true

  # Database
  db_name  = "telecom_db"
  username = "pgadmin"
  password = var.db_password
  port     = 5432

  # Networking
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false

  # Parameter Group
  parameter_group_name = aws_db_parameter_group.postgres16.name

  # Backup
  backup_retention_period = 35          # 35 days (max for PITR)
  backup_window           = "02:00-03:00"
  copy_tags_to_snapshot   = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "prod-postgres-final-${formatdate("YYYY-MM-DD", timestamp())}"

  # Maintenance
  maintenance_window          = "sun:04:00-sun:05:00"
  auto_minor_version_upgrade  = false   # Controlled upgrades in prod
  allow_major_version_upgrade = false

  # Monitoring
  monitoring_interval             = 60  # Enhanced Monitoring every 60 sec
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn
  performance_insights_enabled    = true
  performance_insights_retention_period = 731  # 2 years
  performance_insights_kms_key_id = aws_kms_key.rds.arn
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  # Protection
  deletion_protection = true

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [password, latest_restorable_time]
  }

  tags = {
    Name          = "prod-telecom-postgres"
    BackupPolicy  = "35-day-retention"
    CostCenter    = "DBA-PROD-001"
  }

  depends_on = [
    aws_db_subnet_group.main,
    aws_security_group.rds,
    aws_iam_role_policy_attachment.rds_monitoring,
  ]
}

# --- Read Replica ---
resource "aws_db_instance" "postgres_replica" {
  identifier          = "prod-telecom-postgres-read"
  replicate_source_db = aws_db_instance.postgres.identifier
  instance_class      = "db.r6g.xlarge"
  storage_encrypted   = true
  kms_key_id          = aws_kms_key.rds.arn
  publicly_accessible = false
  multi_az            = false

  performance_insights_enabled    = true
  performance_insights_retention_period = 731
  monitoring_interval             = 60
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn

  tags = { Name = "prod-telecom-postgres-read-replica" }
}

# --- CloudWatch Alarms ---
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "rds-prod-postgres-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "RDS CPU > 80% for 15 minutes"
  alarm_actions       = [var.sns_topic_arn]
  dimensions          = { DBInstanceIdentifier = aws_db_instance.postgres.identifier }
}

resource "aws_cloudwatch_metric_alarm" "free_storage" {
  alarm_name          = "rds-prod-postgres-storage-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 50000000000   # 50 GB in bytes
  alarm_description   = "RDS Free Storage < 50GB"
  alarm_actions       = [var.sns_topic_arn]
  dimensions          = { DBInstanceIdentifier = aws_db_instance.postgres.identifier }
}

resource "aws_cloudwatch_metric_alarm" "replica_lag" {
  alarm_name          = "rds-prod-postgres-replica-lag"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "ReplicaLag"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Average"
  threshold           = 30            # 30 seconds
  alarm_description   = "Read Replica lag > 30 seconds"
  alarm_actions       = [var.sns_topic_arn]
  dimensions          = { DBInstanceIdentifier = aws_db_instance.postgres_replica.identifier }
}
```

---

### Q11. How do you enable PITR (Point-In-Time Recovery)?

**Answer:** PITR is enabled automatically when `backup_retention_period > 0`.

```hcl
resource "aws_db_instance" "postgres" {
  backup_retention_period = 35    # Enables PITR for last 35 days
  backup_window           = "02:00-03:00"  # Preferred backup window (UTC)
  # RDS automatically takes daily snapshots + continuous WAL archiving
  # You can restore to any second within the retention period
}

# To restore via AWS CLI (not Terraform — one-time operation):
# aws rds restore-db-instance-to-point-in-time \
#   --source-db-instance-identifier prod-postgres \
#   --target-db-instance-identifier prod-postgres-pitr \
#   --restore-time "2025-02-23T10:30:00Z"
```

---

## Q12. How do you rotate the master password securely?

```hcl
# APPROACH 1: AWS Secrets Manager Auto-Rotation (Best Practice)
resource "aws_db_instance" "postgres" {
  manage_master_user_password   = true        # AWS manages the password!
  master_user_secret_kms_key_id = aws_kms_key.rds.arn
  # AWS automatically stores password in Secrets Manager
  # and can auto-rotate on schedule
}

# APPROACH 2: Manual rotation with Terraform
# Step 1: Update password in Secrets Manager
# Step 2: terraform apply (Terraform updates RDS password)
# Step 3: Update application connection strings

# APPROACH 3: Ignore password in Terraform, rotate externally
resource "aws_db_instance" "postgres" {
  password = var.db_password
  lifecycle {
    ignore_changes = [password]  # Terraform won't revert password changes
  }
}
# Then rotate via: aws rds modify-db-instance --master-user-password NewP@ss123
```

---

## Q13. How do you perform zero-downtime version upgrade?

**Answer:**

```
  MINOR VERSION UPGRADE (e.g., 16.1 → 16.3):
  ├── Set: auto_minor_version_upgrade = true (or apply manually during maintenance)
  ├── Multi-AZ: Standby upgraded first → failover → primary upgraded
  ├── Downtime: ~30-60 seconds during failover
  └── In Terraform: change engine_version and apply

  MAJOR VERSION UPGRADE (e.g., 14.x → 16.x):
  ├── Step 1: Take manual snapshot (safety net)
  ├── Step 2: Test upgrade on a restored copy first!
  ├── Step 3: Set allow_major_version_upgrade = true in Terraform
  ├── Step 4: Change engine_version = "16.3"
  ├── Step 5: terraform apply (triggers in-place upgrade)
  ├── Step 6: Verify application compatibility
  ├── Downtime: 10-30 minutes (depends on DB size)
  └── Rollback: Restore from pre-upgrade snapshot
```

```hcl
# Major version upgrade
resource "aws_db_instance" "postgres" {
  engine_version              = "16.3"             # Changed from 14.8
  allow_major_version_upgrade = true               # Required for major upgrades
  parameter_group_name        = aws_db_parameter_group.postgres16.name  # New PG family!
  apply_immediately           = false              # Apply during maintenance window
}
```

---
---

# SECTION 3: AWS AURORA — DEEP DIVE

---

## Q14. What is the difference between RDS and Aurora?

| Feature | RDS | Aurora |
|---------|-----|--------|
| **Storage** | EBS volumes (gp2/gp3/io1) | Distributed storage (6 copies across 3 AZs) |
| **Failover** | 60-120 seconds | ~30 seconds (faster DNS update) |
| **Replication** | Async (read replicas) | Sync within cluster (microsecond lag) |
| **Max Replicas** | 5 read replicas | 15 read replicas |
| **Storage Scaling** | Manual (or auto-scaling) | Automatic (10GB → 128TB, no pre-provisioning) |
| **Backtrack** | Not available | Yes (rewind to point in time without restore) |
| **Global DB** | Cross-region read replica | True global database (< 1 sec replication) |
| **Endpoints** | One per instance | Writer endpoint + Reader endpoint + Custom |
| **Serverless** | Not available | Aurora Serverless v2 |
| **Cost** | Lower | ~20% more than RDS |
| **Performance** | Standard | 3-5x faster than standard PostgreSQL (per AWS) |

```
  AURORA ARCHITECTURE:
  ┌──────────────────────────────────────────────────┐
  │                AURORA CLUSTER                     │
  │                                                   │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐     │
  │  │  WRITER  │   │ READER 1 │   │ READER 2 │     │
  │  │ Instance │   │ Instance │   │ Instance │     │
  │  │  (AZ-1a) │   │  (AZ-1b) │   │  (AZ-1c) │     │
  │  └────┬─────┘   └────┬─────┘   └────┬─────┘     │
  │       │               │               │           │
  │  ┌────┴───────────────┴───────────────┴────┐     │
  │  │    SHARED DISTRIBUTED STORAGE LAYER      │     │
  │  │   6 copies of data across 3 AZs          │     │
  │  │   Auto-scaling: 10GB → 128TB             │     │
  │  │   Self-healing: bad blocks auto-repaired │     │
  │  └─────────────────────────────────────────┘     │
  │                                                   │
  │  Writer Endpoint: prod-aurora.cluster-xxx.region.rds.amazonaws.com  │
  │  Reader Endpoint: prod-aurora.cluster-ro-xxx.region.rds.amazonaws.com │
  └──────────────────────────────────────────────────┘
```

---

## Q15. Complete Production Aurora PostgreSQL with Terraform

```hcl
# ============================================================
# PRODUCTION AURORA POSTGRESQL CLUSTER — TERRAFORM
# ============================================================

# --- Aurora Cluster ---
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "prod-telecom-aurora"

  # Engine
  engine         = "aurora-postgresql"
  engine_version = "16.1"
  engine_mode    = "provisioned"      # or "serverless" for Serverless v2

  # Database
  database_name   = "telecom_db"
  master_username = "pgadmin"
  master_password = var.db_password
  port            = 5432

  # Storage
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  # Networking
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  # Backup
  backup_retention_period      = 35
  preferred_backup_window      = "02:00-03:00"
  copy_tags_to_snapshot        = true
  skip_final_snapshot          = false
  final_snapshot_identifier    = "aurora-final-${formatdate("YYYY-MM-DD", timestamp())}"

  # Maintenance
  preferred_maintenance_window = "sun:04:00-sun:05:00"

  # Parameter Group
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.aurora_pg16.name

  # Backtrack (Aurora-specific — rewind without restore!)
  backtrack_window = 86400            # 24 hours backtrack window

  # Logging
  enabled_cloudwatch_logs_exports = ["postgresql"]

  # Protection
  deletion_protection = true

  # Serverless v2 scaling (if using serverless instances)
  serverlessv2_scaling_configuration {
    min_capacity = 2.0    # 2 ACUs minimum
    max_capacity = 64.0   # 64 ACUs maximum
  }

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [master_password]
  }

  tags = { Name = "prod-telecom-aurora-cluster" }
}

# --- Aurora Cluster Parameter Group ---
resource "aws_rds_cluster_parameter_group" "aurora_pg16" {
  name_prefix = "prod-aurora-pg16-"
  family      = "aurora-postgresql16"

  parameter { name = "shared_preload_libraries" value = "pg_stat_statements" apply_method = "pending-reboot" }
  parameter { name = "log_min_duration_statement" value = "1000" apply_method = "immediate" }
  parameter { name = "rds.logical_replication" value = "1" apply_method = "pending-reboot" }

  lifecycle { create_before_destroy = true }
}

# --- Writer Instance ---
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "aurora-writer"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.2xlarge"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  publicly_accessible  = false
  monitoring_interval  = 60
  monitoring_role_arn  = aws_iam_role.rds_monitoring.arn

  performance_insights_enabled    = true
  performance_insights_retention_period = 731

  tags = { Name = "aurora-writer", Role = "writer" }
}

# --- Reader Instances (2 replicas for HA + read scaling) ---
resource "aws_rds_cluster_instance" "readers" {
  count              = 2
  identifier         = "aurora-reader-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.xlarge"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version

  publicly_accessible  = false
  monitoring_interval  = 60
  monitoring_role_arn  = aws_iam_role.rds_monitoring.arn

  performance_insights_enabled = true

  tags = { Name = "aurora-reader-${count.index + 1}", Role = "reader" }
}

# --- Auto Scaling for Read Replicas ---
resource "aws_appautoscaling_target" "aurora_replicas" {
  service_namespace  = "rds"
  scalable_dimension = "rds:cluster:ReadReplicaCount"
  resource_id        = "cluster:${aws_rds_cluster.aurora.id}"
  min_capacity       = 2
  max_capacity       = 5
}

resource "aws_appautoscaling_policy" "aurora_cpu_scaling" {
  name               = "aurora-cpu-auto-scaling"
  service_namespace  = aws_appautoscaling_target.aurora_replicas.service_namespace
  scalable_dimension = aws_appautoscaling_target.aurora_replicas.scalable_dimension
  resource_id        = aws_appautoscaling_target.aurora_replicas.resource_id
  policy_type        = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "RDSReaderAverageCPUUtilization"
    }
    target_value       = 60.0    # Scale out when CPU > 60%
    scale_in_cooldown  = 300
    scale_out_cooldown = 300
  }
}
```

---

## Q16. What is Aurora Global Database?

**Answer:** Aurora Global Database enables a single Aurora database to span multiple AWS regions, providing cross-region disaster recovery with < 1 second replication lag.

```hcl
# Primary cluster (ap-south-1)
resource "aws_rds_global_cluster" "global" {
  global_cluster_identifier = "telecom-global"
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  storage_encrypted         = true
}

resource "aws_rds_cluster" "primary" {
  provider                  = aws.mumbai
  cluster_identifier        = "aurora-primary-mumbai"
  global_cluster_identifier = aws_rds_global_cluster.global.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  master_username           = "pgadmin"
  master_password           = var.db_password
  # ... other settings
}

# Secondary cluster (us-east-1) — read-only, automatic replication
resource "aws_rds_cluster" "secondary" {
  provider                  = aws.virginia
  cluster_identifier        = "aurora-secondary-virginia"
  global_cluster_identifier = aws_rds_global_cluster.global.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.1"
  # No master_username/password — inherited from primary
}
```

---
---

# SECTION 4: GCP CLOUD SQL — DEEP DIVE

---

## Q17. Complete Production GCP Cloud SQL PostgreSQL with Terraform

```hcl
# ============================================================
# GCP CLOUD SQL POSTGRESQL — TERRAFORM
# ============================================================

terraform {
  required_providers {
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
  backend "gcs" {
    bucket = "telecom-terraform-state"
    prefix = "cloudsql/production"
  }
}

provider "google" {
  project = "telecom-prod-project"
  region  = "asia-south1"
}

# --- Cloud SQL Instance ---
resource "google_sql_database_instance" "postgres" {
  name                = "prod-telecom-postgres"
  database_version    = "POSTGRES_16"
  region              = "asia-south1"
  deletion_protection = true

  settings {
    tier              = "db-custom-8-32768"     # 8 vCPU, 32 GB RAM
    availability_type = "REGIONAL"               # HA across 2 zones
    disk_type         = "PD_SSD"
    disk_size         = 500                      # GB
    disk_autoresize   = true
    disk_autoresize_limit = 1000                 # Max 1TB

    # Backup
    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "02:00"   # UTC
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 35
        retention_unit   = "COUNT"
      }
    }

    # Networking (Private IP only)
    ip_configuration {
      ipv4_enabled                                  = false   # No public IP!
      private_network                               = var.vpc_network
      enable_private_path_for_google_cloud_services = true
    }

    # Maintenance
    maintenance_window {
      day          = 7      # Sunday
      hour         = 4      # 4 AM UTC
      update_track = "stable"
    }

    # Database Flags (Parameter Tuning)
    database_flags { name = "max_connections"            value = "500" }
    database_flags { name = "shared_buffers"             value = "8192MB" }
    database_flags { name = "effective_cache_size"       value = "24576MB" }
    database_flags { name = "work_mem"                   value = "64MB" }
    database_flags { name = "log_min_duration_statement" value = "1000" }
    database_flags { name = "log_connections"            value = "on" }
    database_flags { name = "log_disconnections"         value = "on" }
    database_flags { name = "cloudsql.logical_decoding"  value = "on" }

    # Insights (Performance Monitoring)
    insights_config {
      query_insights_enabled  = true
      query_plans_per_minute  = 5
      query_string_length     = 4500
      record_application_tags = true
      record_client_address   = true
    }

    user_labels = {
      environment = "production"
      team        = "dba"
      managed_by  = "terraform"
    }
  }
}

# --- Database ---
resource "google_sql_database" "telecom" {
  name     = "telecom_db"
  instance = google_sql_database_instance.postgres.name
}

# --- Users ---
resource "google_sql_user" "admin" {
  name     = "pgadmin"
  instance = google_sql_database_instance.postgres.name
  password = var.db_password
}

resource "google_sql_user" "app_user" {
  name     = "telecom_app"
  instance = google_sql_database_instance.postgres.name
  password = var.app_password
}

# --- Read Replica ---
resource "google_sql_database_instance" "read_replica" {
  name                 = "prod-telecom-postgres-replica"
  database_version     = "POSTGRES_16"
  region               = "asia-south1"
  master_instance_name = google_sql_database_instance.postgres.name

  replica_configuration {
    failover_target = false
  }

  settings {
    tier            = "db-custom-4-16384"   # Smaller than primary
    disk_type       = "PD_SSD"
    disk_autoresize = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_network
    }

    insights_config {
      query_insights_enabled = true
    }
  }

  deletion_protection = false   # Replicas can be destroyed
}

# --- Cloud SQL Monitoring (Cloud Monitoring alert) ---
resource "google_monitoring_alert_policy" "cpu_alert" {
  display_name = "Cloud SQL CPU > 80%"
  combiner     = "OR"

  conditions {
    display_name = "CPU utilization high"
    condition_threshold {
      filter          = "resource.type = \"cloudsql_database\" AND metric.type = \"cloudsql.googleapis.com/database/cpu/utilization\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8
    }
  }

  notification_channels = [var.notification_channel_id]
}
```

---
---

# SECTION 5: ADVANCED PRODUCTION SCENARIOS

---

## Q18. Production RDS suddenly deleted. How do you prevent and recover?

**Answer — 3 Prevention Layers + 2 Recovery Methods:**

```
  PREVENTION:
  Layer 1: deletion_protection = true (AWS API level)
           → AWS will refuse to delete the instance
  Layer 2: lifecycle { prevent_destroy = true } (Terraform level)
           → terraform destroy will ERROR before even calling AWS
  Layer 3: IAM policy (permission level)
           → Deny rds:DeleteDBInstance for all but break-glass role
  
  RECOVERY (if all 3 failed):
  Method 1: Restore from automated snapshot (within retention period)
            → aws rds restore-db-instance-from-db-snapshot
  Method 2: PITR (restore to the exact second before deletion)
            → aws rds restore-db-instance-to-point-in-time
```

---

## Q19. How do you manage Dev, QA, Staging, Prod with Terraform?

**Answer — Two approaches:**

### Approach 1: Separate Directories (Recommended)

```
  environments/
  ├── dev/
  │   ├── main.tf           → module "rds" { source = "../../modules/rds" }
  │   ├── terraform.tfvars  → instance_class = "db.t3.medium"
  │   └── backend.tf        → key = "rds/dev/terraform.tfstate"
  ├── qa/
  │   ├── main.tf
  │   ├── terraform.tfvars  → instance_class = "db.r6g.large"
  │   └── backend.tf        → key = "rds/qa/terraform.tfstate"
  └── prod/
      ├── main.tf
      ├── terraform.tfvars  → instance_class = "db.r6g.2xlarge"
      └── backend.tf        → key = "rds/prod/terraform.tfstate"

  # Each environment has its own state file — completely isolated!
```

### Approach 2: Workspaces

```bash
terraform workspace new dev
terraform workspace new qa
terraform workspace new prod

terraform workspace select prod
terraform apply -var-file=prod.tfvars
```

---

## Q20. How do you handle secrets (passwords) in Terraform?

```hcl
# ============================================================
# APPROACH 1: AWS Secrets Manager (Best Practice)
# ============================================================
data "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "prod/rds/master-password"
}

resource "aws_db_instance" "postgres" {
  password = data.aws_secretsmanager_secret_version.db_pass.secret_string
}

# ============================================================
# APPROACH 2: AWS-Managed Password (Simplest — RDS manages it)
# ============================================================
resource "aws_db_instance" "postgres" {
  manage_master_user_password   = true
  master_user_secret_kms_key_id = aws_kms_key.rds.arn
}

# ============================================================
# APPROACH 3: GCP Secret Manager
# ============================================================
data "google_secret_manager_secret_version" "db_pass" {
  secret = "prod-cloudsql-password"
}

resource "google_sql_database_instance" "postgres" {
  # use: data.google_secret_manager_secret_version.db_pass.secret_data
}

# ============================================================
# NEVER DO THIS:
# ============================================================
# password = "MyP@ssword123"        ← NEVER hardcode!
# password = var.db_password         ← OK only if passed via CI/CD env vars
```

---
---

# SECTION 6: DEEP TECHNICAL COMPARISON TABLE

---

## Q21. RDS vs Aurora vs GCP Cloud SQL — Complete Comparison

| Feature | AWS RDS | AWS Aurora | GCP Cloud SQL |
|---------|---------|------------|---------------|
| **Storage** | EBS (gp2/gp3/io1) | Distributed (6 copies, 3 AZs) | Persistent Disk (PD-SSD) |
| **Max Storage** | 64 TB | 128 TB (auto-scales) | 64 TB |
| **Failover Time** | 60-120 sec | ~30 sec | ~60 sec |
| **Read Replicas** | Up to 5 | Up to 15 | Up to 10 |
| **Replica Lag** | Seconds to minutes | Milliseconds | Seconds to minutes |
| **Global DB** | Cross-region read replica | True global DB (< 1s lag) | Cross-region read replica |
| **Backtrack** | No | Yes (rewind without restore) | No |
| **Serverless** | No | Serverless v2 | No |
| **Auto Storage Scaling** | Yes (max_allocated_storage) | Automatic (always) | Yes (disk_autoresize) |
| **Performance** | Standard | 3-5x faster (per AWS) | Standard |
| **Monitoring** | Performance Insights + CloudWatch | Performance Insights + CloudWatch | Query Insights + Cloud Monitoring |
| **Encryption** | KMS | KMS | CMEK or Google-managed |
| **IAM DB Auth** | Yes | Yes | Yes (Cloud IAM) |
| **Parameter Tuning** | Parameter Groups | Cluster Parameter Groups | Database Flags |
| **Cost** | Medium | ~20% more than RDS | Medium |
| **Best For** | Standard OLTP, cost-sensitive | High-performance, global, serverless | GCP-native apps, multi-cloud |

---
---

# SECTION 7: MOCK INTERVIEW RAPID FIRE (20 Questions)

---

## Q22–Q41: Rapid Fire with Concise Answers

**Q22. What happens if the Terraform state file is corrupted?**
Restore from S3 versioning (always enable versioning on state bucket). If unavailable, delete state and re-import all resources using `terraform import`.

**Q23. How do you recover Terraform state?**
S3 bucket versioning → download previous version. Or `terraform import resource_type.name resource_id` for each resource.

**Q24. Can multiple engineers run Terraform simultaneously?**
Not safely without state locking. Use DynamoDB table for S3 backend (or GCS bucket locking). Only one `terraform apply` can run at a time.

**Q25. How do you implement CI/CD for Terraform?**
Jenkins/GitHub Actions pipeline: checkout → `terraform init` → `terraform plan` (save output) → manual approval for prod → `terraform apply`. Store state in remote backend.

**Q26. How to do blue/green DB deployment?**
Create new RDS instance with new config → migrate data → switch application endpoint → verify → destroy old instance. Use `create_before_destroy` lifecycle.

**Q27. Explain WAL in PostgreSQL on RDS.**
WAL (Write-Ahead Log) records all changes before writing to data files. RDS uses WAL for: PITR (continuous WAL archiving to S3), replication (streaming WAL to read replicas), crash recovery. Enable `rds.logical_replication` in parameter group for logical replication.

**Q28. How does replication slot work in RDS?**
Replication slots ensure WAL segments aren't recycled until consumed by replicas. In RDS: physical slots for read replicas (managed by AWS), logical slots for CDC tools (Debezium, etc.). Monitor with `pg_replication_slots` view.

**Q29. How to monitor replication lag?**
CloudWatch metric: `ReplicaLag` (seconds behind primary). SQL query on replica: `SELECT now() - pg_last_xact_replay_timestamp() AS lag;`. Set CloudWatch alarm for lag > 30 seconds.

**Q30. How to tune Aurora PostgreSQL?**
Use cluster parameter group for shared settings. Key parameters: `shared_buffers`, `effective_cache_size`, `max_connections`, `work_mem`. Enable Performance Insights for query-level analysis. Use `pg_stat_statements` for slow query identification.

**Q31. How to tune Cloud SQL PostgreSQL?**
Use database_flags in Terraform. Key flags: `shared_buffers`, `effective_cache_size`, `work_mem`, `max_connections`. Enable Query Insights. Use `pg_stat_statements` extension.

**Q32. Parameter group tuning via Terraform?**
Create custom parameter group resource → set parameters → reference in RDS instance. Use `apply_method = "immediate"` for runtime params, `"pending-reboot"` for restart-required params.

**Q33. How does RDS failover work internally?**
Multi-AZ: AWS detects primary failure → promotes standby → updates DNS CNAME to point to new primary → application reconnects (60-120 sec). No data loss (synchronous replication).

**Q34. What happens during AZ failure?**
Multi-AZ RDS: automatic failover to standby in surviving AZ. Aurora: promotes one of the read replicas (fastest ~30 sec). Application must handle brief connection interruption and retry.

**Q35. How do you enable encryption on RDS?**
Set `storage_encrypted = true` and provide `kms_key_id` at creation time. Cannot enable encryption on existing unencrypted instance — must snapshot, copy with encryption, restore from encrypted snapshot.

**Q36. What is RDS option group?**
Option groups provide additional features for specific engines. Examples: Oracle TDE, SQL Server TDE, MySQL memcached. PostgreSQL has minimal option group usage — most config is via parameter groups.

**Q37. How does Aurora storage auto-scaling work?**
Aurora storage automatically grows in 10GB increments as data grows. No pre-provisioning needed. Grows from 10GB to 128TB. You never set `allocated_storage` — Aurora manages it. Storage is only billed for what you use.

**Q38. Difference between Aurora MySQL and Aurora PostgreSQL?**
Same distributed storage engine. MySQL version supports parallel query, Aurora Backtrack. PostgreSQL version supports Babelfish (SQL Server compatibility), logical replication. Both support Global Database, Serverless v2.

**Q39. How to migrate from on-prem PostgreSQL to Cloud SQL?**
Options: 1) pg_dump/pg_restore (small DBs). 2) GCP Database Migration Service (DMS) for continuous replication. 3) Streaming replication to Cloud SQL replica, then promote. 4) Use Bucardo or Londiste for logical replication.

**Q40. How do you enable private IP on Cloud SQL?**
Set `ipv4_enabled = false` and provide `private_network = google_compute_network.vpc.id` in Terraform. Requires VPC peering to Google's service network (automatic when using private IP).

**Q41. How to scale Cloud SQL vertically?**
Change `tier` in Terraform (e.g., `db-custom-4-16384` → `db-custom-8-32768`). Apply with `terraform apply`. Causes brief downtime (~1-5 minutes). HA instances: standby upgraded first, failover, then primary upgraded.

---
---

# SECTION 8: TERRAFORM BEST PRACTICES FOR DBA

---

## Q42. 15 Best Practices for Database Infrastructure with Terraform

1. **Use modules** — Write once, deploy to dev/qa/staging/prod
2. **Remote state with locking** — S3 + DynamoDB (AWS) or GCS (GCP)
3. **Separate state per environment** — dev, qa, prod have isolated state files
4. **Never hardcode secrets** — Use Secrets Manager / Secret Manager / Vault
5. **Enable encryption everywhere** — `storage_encrypted = true`, KMS keys, CMEK
6. **Private networking only** — `publicly_accessible = false`, private subnets
7. **Deletion protection** — 3 layers: AWS protection + lifecycle + IAM
8. **Tag everything** — Environment, Team, ManagedBy, CostCenter, Project
9. **Enable monitoring** — Performance Insights, CloudWatch/Cloud Monitoring alarms
10. **Use parameter groups/database flags** — Version-control your database tuning
11. **Plan before apply** — Always review `terraform plan` output before production changes
12. **Git-based workflow** — All .tf files in Git, PRs for review, CI/CD for deployment
13. **State file security** — Encrypt state at rest (S3 encryption), restrict bucket access
14. **Version pin providers** — `version = "~> 5.0"` prevents unexpected provider upgrades
15. **Test upgrades on copy first** — Restore snapshot → test upgrade → then apply to production

---

*Terraform + IaaS for DBA — Complete Interview Guide*
*AWS RDS • Aurora • GCP Cloud SQL • Terraform Best Practices*
*75+ Questions with Detailed Answers, Production Terraform Code & Architecture Diagrams*
