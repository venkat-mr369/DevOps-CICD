**case-study style Terraform interview Q\&A**. 

---

# 🔹 Terraform Case Study Interview Questions & Solutions

---

## **Case 1: Multi-Region High Availability**

**Scenario:** Your company wants to deploy an application on **AWS** across 2 regions (us-east-1 and us-west-2) with a failover strategy. You must design this using Terraform.

**Q:** How would you design the Terraform configuration for multi-region HA?

**Solution:**

* Use **provider aliasing** for multiple regions.
* Example:

  ```hcl
  provider "aws" {
    region = "us-east-1"
  }

  provider "aws" {
    alias  = "west"
    region = "us-west-2"
  }

  resource "aws_s3_bucket" "east" {
    bucket = "my-ha-bucket-east"
    provider = aws
  }

  resource "aws_s3_bucket" "west" {
    bucket = "my-ha-bucket-west"
    provider = aws.west
  }
  ```
* For HA:

  * Use **Route53 failover records**.
  * Ensure **state is centralized** (S3 + DynamoDB locking).
  * Use **Terraform workspaces** for dev/test/prod separation.

---

## **Case 2: Remote State & Team Collaboration**

**Scenario:** Your team of 20 developers is working on Terraform. You must ensure **no state corruption** and enable collaboration.

**Q:** How would you set up Terraform state management?

**Solution:**

* Store state in **remote backend** (S3, GCS, or Azure Blob).
* Enable **locking** (DynamoDB for AWS, Cloud Storage Locking for GCP).
* Example:

  ```hcl
  terraform {
    backend "s3" {
      bucket         = "my-terraform-state"
      key            = "infra/prod/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "terraform-locks"
      encrypt        = true
    }
  }
  ```
* Enforce state management rules:

  * No local `tfstate`.
  * CI/CD pipelines run Terraform, not individuals.

---

## **Case 3: Terraform + CI/CD Pipelines**

**Scenario:** You need to integrate Terraform with GitHub Actions for automated deployments.

**Q:** How do you set up CI/CD with Terraform?

**Solution:**

* Workflow:

  1. Developer commits `.tf` → GitHub PR.
  2. CI runs `terraform fmt`, `terraform validate`, `terraform plan`.
  3. On approval, `terraform apply`.
* Example GitHub Actions job:

  ```yaml
  jobs:
    terraform:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - uses: hashicorp/setup-terraform@v2
        - run: terraform init
        - run: terraform validate
        - run: terraform plan -out=tfplan
        - run: terraform apply -auto-approve tfplan
  ```
* Secrets (AWS creds, etc.) → GitHub Secrets.
* Use **separate workspaces** or **state files per env**.

---

## **Case 4: Handling Sensitive Data**

**Scenario:** You need to store database credentials in Terraform without exposing them in code or state.

**Q:** How do you manage secrets securely in Terraform?

**Solution:**

* Use **Vault Provider** or **Cloud Secret Managers**.
* Example with AWS Secrets Manager:

  ```hcl
  data "aws_secretsmanager_secret_version" "db_pass" {
    secret_id = "prod/db_password"
  }

  resource "aws_db_instance" "db" {
    identifier = "prod-db"
    username   = "admin"
    password   = data.aws_secretsmanager_secret_version.db_pass.secret_string
  }
  ```
* Also:

  * Set `sensitive = true` in output.
  * Encrypt tfstate backend.

---

## **Case 5: Large Scale Infrastructure (1000+ Resources)**

**Scenario:** Your infra is huge (VPCs, EC2s, RDS, etc.). Running `terraform plan` takes 10+ minutes.

**Q:** How do you optimize Terraform for large-scale deployments?

**Solution:**

* Split infra into **multiple states**:

  * `networking` (VPC, subnets, SGs).
  * `compute` (EC2, ASG, ELB).
  * `databases`.
* Use **modules** for reusability.
* Run **Terraform apply in parallel** with `-target`.
* Enable **caching in CI/CD** to avoid repeated downloads of providers/modules.

---

## **Case 6: Disaster Recovery**

**Scenario:** Your Terraform state file in S3 is deleted accidentally.

**Q:** How would you recover?

**Solution:**

* If versioning enabled in S3 → Restore previous version.
* If no backup → Import resources back into new state (`terraform import`).
* Best Practice:

  * Always enable **S3 versioning**.
  * Enable **remote state snapshots**.
  * Store state backups in **different region**.

---

## **Case 7: Terraform & Hybrid Cloud**

**Scenario:** Your infra is partly on-prem (VMware vSphere) and partly in AWS.

**Q:** Can Terraform manage both, and how?

**Solution:**

* Yes, Terraform supports multiple providers.
* Example:

  ```hcl
  provider "vsphere" {
    user           = "administrator@vsphere.local"
    password       = var.vsphere_password
    vsphere_server = "vsphere.local"
    allow_unverified_ssl = true
  }

  provider "aws" {
    region = "us-east-1"
  }

  resource "vsphere_virtual_machine" "vm" {
    name   = "onprem-vm"
    cpu    = 2
    memory = 4096
  }

  resource "aws_instance" "ec2" {
    ami           = "ami-123456"
    instance_type = "t3.medium"
  }
  ```
* Use **data sources** to connect on-prem with AWS resources.

---

## **Case 8: Policy Enforcement**

**Scenario:** Developers are creating resources with Terraform, but you must enforce **compliance rules** (e.g., no public S3 buckets).

**Q:** How would you enforce policies in Terraform?

**Solution:**

* Use **Sentinel policies** (HashiCorp Enterprise) or **OPA (Open Policy Agent)**.
* Example Sentinel policy (deny public S3):

  ```hcl
  import "tfplan/v2" as tfplan
  s3_buckets = filter tfplan.resources as r {
    r.type is "aws_s3_bucket"
  }
  all s3_buckets as bucket {
    bucket.values.acl is not "public-read"
  }
  ```
* For OSS users:

  * Use `terraform validate` with custom linters.
  * Integrate OPA with CI/CD.

---

## **Case 9: Blue-Green Deployment**

**Scenario:** You need zero-downtime deployments for an app running on EC2.

**Q:** How do you design blue-green deployment with Terraform?

**Solution:**

* Approach:

  * Create **two ASGs/ELBs** (blue & green).
  * Use Terraform to flip DNS (Route53) between them.
* Example:

  ```hcl
  resource "aws_route53_record" "app" {
    zone_id = "Z12345"
    name    = "app.example.com"
    type    = "CNAME"
    ttl     = 60
    records = [aws_lb.green.dns_name] # switch between green/blue
  }
  ```
* Deployment flow:

  1. Deploy new version to “green”.
  2. Run health checks.
  3. Switch DNS to green.
  4. Destroy “blue” after validation.

---

## **Case 10: Interview Challenge**

**Q:** Your CTO asks you: *“If Terraform is down (e.g., SaaS Cloud services unavailable), how will you manage infra?”*

**Solution:**

* If using Terraform OSS → unaffected (local execution).
* If using Terraform Cloud/Enterprise:

  * Use **local CLI execution with remote state**.
  * Infra itself is not dependent on Terraform runtime.
* Best Practice:

  * Maintain **local CLI fallback** strategy.
  * Keep **IaC in Git** for reproducibility.

---
