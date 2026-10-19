# Terraform + AWS Interview Cheat Sheet

Short answers with standard snippets. Each ends with a one-liner you can say in the interview.

---

# Part 1: Terraform Core

## 1. `count` vs `for_each`
```hcl
resource "aws_instance" "a" { count = 3 }                          # index: aws_instance.a[0]
resource "aws_instance" "b" { for_each = toset(["web", "app"]) }   # key: aws_instance.b["web"]
```
**One-liner:** `count` is index-based, so removing a middle item shifts and recreates the rest. `for_each` uses stable keys. Use `count` for on/off (`? 1 : 0`) and `for_each` for distinct items.

## 2. State file and why remote state
State maps your config to real resources (IDs, attributes, dependencies). Remote state lets a team share it, with locking, encryption and versioning.
```hcl
terraform {
  backend "s3" {
    bucket  = "my-tfstate"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```
**One-liner:** State is Terraform's source of truth for what it manages. Keep it remote, encrypted and locked, never in Git.

## 3. State locking (S3 + DynamoDB)
```hcl
backend "s3" {
  bucket         = "my-tfstate"
  key            = "prod/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "tf-locks"      # hash key: LockID (String)
}
# Newer Terraform (1.10+): use_lockfile = true  (native S3 locking, no DynamoDB)
```
**One-liner:** Locking stops two applies from writing state at once. If it's stuck, check for a running apply first, then `terraform force-unlock <ID>`.

## 4. `plan` vs `apply` vs `refresh`
- `plan`: shows the diff between config, state and real infra. Changes nothing.
- `apply`: executes the plan.
- `refresh`: syncs state with real infra. Now done with `terraform apply -refresh-only`.

**One-liner:** Plan previews, apply changes, refresh-only updates state without changing infrastructure.

## 5. `locals` vs variables
```hcl
variable "env" { type = string }              # input, set by the caller

locals {
  name_prefix = "app-${var.env}"              # computed internally, not overridable
}
```
**One-liner:** Variables are inputs. Locals are internal named expressions that avoid repetition.

## 6. Variable types and validation
```hcl
variable "vms" {
  type = map(object({
    instance_type = string
    ami_id        = optional(string)          # Terraform 1.3+
  }))
}

variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "stage", "prod"], var.env)
    error_message = "env must be dev, stage or prod."
  }
}
```
**One-liner:** Strong types plus `validation` blocks catch bad input at plan time.

## 7. Handling sensitive values
```hcl
variable "db_password" {
  type      = string
  sensitive = true                            # hidden in CLI output
}

data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db"
}
```
**One-liner:** `sensitive` hides values in output but they are still plaintext in state, so encrypt the backend, restrict access, and pull secrets from Secrets Manager or Vault.

## 8. Data sources vs resources
```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
# use: data.aws_ami.al2023.id
```
**One-liner:** Resources create and manage things. Data sources only read existing things.

---

# Part 2: Terraform Intermediate

## 9. Modules: structure and versioning
```
modules/vm/{main.tf, variables.tf, outputs.tf}
```
```hcl
module "vm" {
  source  = "git::https://github.com/org/tf-modules.git//vm?ref=v1.2.0"
  # or registry: source = "terraform-aws-modules/ec2-instance/aws", version = "~> 5.0"
  env     = var.env
}
```
**One-liner:** Modules are reusable packages with inputs and outputs. Pin versions with a git tag or registry version.

## 10. `lifecycle` rules
```hcl
resource "aws_instance" "vm" {
  lifecycle {
    prevent_destroy       = true              # error on destroy
    create_before_destroy = true              # replace without downtime
    ignore_changes        = [tags, user_data] # ignore external edits
  }
}
```
**One-liner:** Lifecycle rules control replace, destroy and drift behaviour.

## 11. `depends_on` vs implicit dependencies
```hcl
resource "aws_eip" "ip" {
  instance = aws_instance.vm.id               # implicit: reference creates the dependency
}

resource "aws_instance" "app" {
  depends_on = [aws_iam_role_policy.p]        # explicit: no attribute reference exists
}
```
**One-liner:** References create dependencies automatically. Use `depends_on` only for hidden dependencies.

## 12. Dynamic blocks
```hcl
resource "aws_security_group" "sg" {
  name = "web-sg"
  dynamic "ingress" {
    for_each = var.ports                      # e.g. [80, 443]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
    }
  }
}
```
**One-liner:** Dynamic blocks generate repeated nested blocks from a collection.

## 13. Importing existing resources
```hcl
import {                                       # Terraform 1.5+
  to = aws_instance.vm
  id = "i-0abc123def456"
}
```
```bash
terraform plan -generate-config-out=generated.tf
terraform import aws_instance.vm i-0abc123def456    # classic CLI way
```
**One-liner:** Import brings existing infra under Terraform management. You still write or generate the matching config.

## 14. `moved` blocks (refactor without destroying)
```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```
**One-liner:** `moved` renames or relocates resources in state, so Terraform doesn't destroy and recreate them.

## 15. Workspaces vs separate state per environment
```bash
terraform workspace new prod
terraform workspace select prod
```
**One-liner:** Workspaces share the same code and backend with separate state, which is fine for similar environments. For strong isolation (prod vs dev), use separate directories or backends and accounts.

## 16. Detecting and fixing drift
```bash
terraform plan -refresh-only      # shows drift
terraform apply                   # revert infra to match code
terraform apply -refresh-only     # or accept drift into state, then update the code
```
**One-liner:** Run scheduled plans in CI to detect drift. Then either revert the manual change or update the code to match.

## 17. Multiple regions or accounts (provider aliases)
```hcl
provider "aws" { region = "us-east-1" }

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
  assume_role { role_arn = "arn:aws:iam::222222222222:role/tf" }
}

resource "aws_s3_bucket" "dr" {
  provider = aws.dr
  bucket   = "my-dr-bucket"
}
```
**One-liner:** Aliased providers target other regions or accounts. Pass them into modules with `providers = { aws = aws.dr }`.

## 18. State corrupted or lost
```bash
# Restore a previous version from the S3 bucket (versioning enabled)
terraform state pull > backup.tfstate
terraform state push backup.tfstate
# If no backup: rebuild with import blocks / terraform import
```
**One-liner:** Prevention is the answer: versioned, encrypted remote backend plus restricted access. Recovery is restoring a previous version or re-importing.

## 19. `taint` / `-replace`
```bash
terraform apply -replace="aws_instance.vm"      # preferred
terraform taint aws_instance.vm                 # older, deprecated
```
**One-liner:** Use `-replace` to force recreation of one resource, for example a broken instance.

## 20. Terraform in CI/CD safely
```bash
terraform fmt -check && terraform validate
terraform plan -out=tfplan        # on PR; post the plan as a comment
terraform apply tfplan            # on merge, after approval
```
**One-liner:** Plan on PR, approve, apply the saved plan on merge. Use remote state with locking, least-privilege or OIDC credentials, and policy checks.

## 21. Terraform Cloud, Atlantis, Sentinel/OPA
- **Terraform Cloud/Enterprise:** hosted runs, remote state, RBAC, policies.
- **Atlantis:** self-hosted PR automation (`atlantis plan` / `apply` comments).
- **Sentinel / OPA (Conftest):** policy as code, for example "no public S3 buckets".

**One-liner:** These add collaboration, automation and guardrails around plan and apply.

## 22. `terraform_remote_state` vs data sources
```hcl
data "terraform_remote_state" "net" {
  backend = "s3"
  config  = { bucket = "my-tfstate", key = "net/terraform.tfstate", region = "us-east-1" }
}
# data.terraform_remote_state.net.outputs.vpc_id

data "aws_vpc" "main" { tags = { Name = "main" } }   # queries the real VPC
```
**One-liner:** Remote state couples you to another stack's outputs and needs read access to that state. Data sources query the real infrastructure and are looser coupling, so they are generally preferred.

---

# Part 3: AWS Networking

## 23. What makes a subnet public?
A route table entry `0.0.0.0/0 -> igw-xxx`, and instances need a public or Elastic IP.
```hcl
resource "aws_route" "public" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}
```
**One-liner:** A subnet is public only because its route table has a route to an Internet Gateway.

## 24. Security group vs NACL
| | Security Group | NACL |
|---|---|---|
| Level | Instance/ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny, ordered by rule number |

**One-liner:** SGs are stateful instance-level allow lists. NACLs are stateless subnet-level rules, and you must allow return traffic (ephemeral ports).

## 25. VPC endpoints: gateway vs interface
- **Gateway:** S3 and DynamoDB only, route-table based, free.
- **Interface (PrivateLink):** ENI with a private IP in your subnet, for most other services, hourly and data cost.
```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}
```
**One-liner:** Endpoints reach AWS services privately without a NAT or internet gateway.

## 26. VPC Peering vs Transit Gateway
- **Peering:** one-to-one, non-transitive, cheap, CIDRs must not overlap.
- **Transit Gateway:** hub-and-spoke, transitive, scales to many VPCs and on-prem, extra cost.

**One-liner:** Peering for a few VPCs, Transit Gateway for many or hybrid. PrivateLink if you only need to expose one service or CIDRs overlap.

## 27. Route 53 routing policies
Simple, Weighted (canary/A-B), Latency, Failover (health-check based), Geolocation, Geoproximity, Multivalue.

**One-liner:** Pick the policy by goal: weighted for rollouts, latency for speed, failover for DR.

## 28. ALB vs NLB vs GWLB
| | ALB | NLB | GWLB |
|---|---|---|---|
| Layer | 7 (HTTP/S) | 4 (TCP/UDP/TLS) | 3 (IP) |
| Use | Path/host routing, WAF | Ultra-low latency, static IP, PrivateLink | Inline firewalls and appliances |

**One-liner:** ALB for web routing, NLB for raw performance and static IPs, GWLB for third-party security appliances.

## 29. ALB 504: causes and fixes
504 means the target timed out. Check:
- App response time versus the ALB idle timeout (default 60 s)
- Target security group allows the ALB SG
- Slow DB or backend calls
- Targets overloaded (scale out)

**One-liner:** 502 is a bad response, 503 is no healthy targets, 504 is a timeout. For 504, raise the idle timeout or fix the slow backend.

## 30. On-prem to AWS: VPN vs Direct Connect
- **Site-to-Site VPN:** encrypted over the internet, quick setup, variable latency.
- **Direct Connect:** dedicated private link, consistent latency and bandwidth, takes weeks.
- Use both (VPN as backup) for resilience.

**One-liner:** VPN is fast and cheap to start. Direct Connect is for predictable, high-throughput workloads.

---

# Part 4: AWS Compute and Storage

## 31. EC2 purchase options
- **On-Demand:** flexible, full price.
- **Reserved / Savings Plans:** 1 or 3 year commitment, large discount for steady workloads.
- **Spot:** up to roughly 90% off, can be interrupted, for fault-tolerant jobs.
- **Dedicated Hosts:** compliance or licensing.

**One-liner:** Baseline on Savings Plans, burst on On-Demand, batch and stateless on Spot.

## 32. Auto Scaling groups
```hcl
resource "aws_autoscaling_group" "asg" {
  min_size            = 2
  max_size            = 6
  desired_capacity    = 2
  vpc_zone_identifier = var.private_subnet_ids
  health_check_type   = "ELB"
  launch_template {
    id      = aws_launch_template.lt.id
    version = "$Latest"
  }
}
```
Policies: target tracking (for example CPU at 50%), step, scheduled.

**One-liner:** ASGs keep capacity within min and max, replace unhealthy instances, and scale on metrics or schedules.

## 33. EBS vs instance store vs EFS
- **EBS:** persistent block storage, one AZ, snapshots.
- **Instance store:** very fast, ephemeral, lost on stop or terminate.
- **EFS:** shared NFS file system, multi-AZ, many instances at once.

**One-liner:** EBS for a single instance's disk, EFS for shared files, instance store for scratch data.

## 34. S3 storage classes and lifecycle
Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier Instant, Glacier Flexible, Glacier Deep Archive.
```hcl
resource "aws_s3_bucket_lifecycle_configuration" "lc" {
  bucket = aws_s3_bucket.b.id
  rule {
    id     = "archive"
    status = "Enabled"
    filter {}
    transition { days = 30  storage_class = "STANDARD_IA" }
    transition { days = 90  storage_class = "GLACIER" }
    expiration { days = 365 }
  }
}
```
**One-liner:** Lifecycle rules move or expire data automatically to cut cost.

## 35. S3 versioning, replication, Object Lock
- **Versioning:** keeps all versions, protects against accidental delete or overwrite.
- **Replication (CRR/SRR):** async copy to another bucket, needs versioning on both.
- **Object Lock:** WORM (compliance or governance mode) for retention and ransomware protection.

**One-liner:** Versioning is the base. Replication gives DR and compliance copies, and Object Lock makes data immutable.

## 36. Securing an S3 bucket
- Block Public Access on (account and bucket)
- SSE-KMS encryption
- Bucket policy denying non-TLS (`aws:SecureTransport = false`)
- Least-privilege IAM roles, no wildcards
- Versioning, access logs or CloudTrail data events
- VPC endpoint with `aws:SourceVpce` conditions

**One-liner:** Private by default, encrypted, TLS-only, least privilege and logged.

---

# Part 5: AWS IAM and Security

## 37. IAM user vs role vs group vs policy
- **User:** a person or app with long-lived credentials.
- **Group:** collection of users sharing policies.
- **Role:** assumable identity with temporary credentials.
- **Policy:** the JSON permissions document.

**One-liner:** Prefer roles (temporary credentials) over users with access keys. Humans should use IAM Identity Center.

## 38. Identity-based vs resource-based policies
- **Identity-based:** attached to a user, group or role ("what can this identity do").
- **Resource-based:** attached to a resource such as S3, KMS or SQS, with a `Principal` ("who can access this resource").

**One-liner:** Within one account, either policy can grant access. Cross-account needs both sides to allow it.

## 39. How `sts:AssumeRole` works
```hcl
# Trust policy on the target role
assume_role_policy = jsonencode({
  Version = "2012-10-17"
  Statement = [{
    Effect    = "Allow"
    Principal = { AWS = "arn:aws:iam::111111111111:root" }
    Action    = "sts:AssumeRole"
  }]
})
```
```bash
aws sts assume-role --role-arn arn:aws:iam::222222222222:role/target --role-session-name test
```
**One-liner:** The caller needs permission to assume the role, and the role's trust policy must trust the caller. STS returns short-lived credentials.

## 40. SCPs vs IAM policies
- **SCP:** Organizations guardrail that sets the maximum permissions for accounts or OUs. It grants nothing by itself.
- **IAM policy:** actually grants permissions inside the account.

**One-liner:** Effective access is the intersection of the SCP and IAM. SCPs don't affect the management account.

## 41. KMS: CMK vs AWS-managed keys, key policy vs IAM
- **AWS-managed:** free, automatic rotation, no control over the key policy.
- **Customer-managed (CMK):** you control the policy, rotation, grants and cross-account use (monthly fee).
- Key policy is mandatory. IAM policies only work if the key policy allows IAM to control access.

**One-liner:** Use CMKs when you need control, cross-account access or audit. The key policy is the root of trust.

## 42. Secrets Manager vs SSM Parameter Store
| | Secrets Manager | Parameter Store |
|---|---|---|
| Rotation | Built-in (Lambda) | No native rotation |
| Cost | Per secret per month | Standard tier free |
| Cross-account sharing | Yes | Limited (advanced tier) |

**One-liner:** Secrets Manager for credentials that need rotation. Parameter Store for config and simple secrets.

## 43. Auditing access
- **CloudTrail:** who did what (API calls).
- **AWS Config:** resource configuration history and compliance rules.
- **GuardDuty:** threat detection.
- **IAM Access Analyzer:** finds external and unused access.
- **Security Hub:** aggregates findings.

**One-liner:** CloudTrail for who, Config for what changed, GuardDuty for threats.

## 44. IAM policy evaluation logic
1. Default is **implicit deny**
2. Any **explicit deny** wins immediately
3. Otherwise an **allow** (identity or resource policy) permits it
4. SCPs and permission boundaries must also allow it

**One-liner:** Explicit deny beats allow, and allow beats implicit deny.

---

# Part 6: AWS Containers and Operations

## 45. EKS vs ECS vs Fargate
- **EKS:** managed Kubernetes, portable, more complex.
- **ECS:** AWS-native orchestration, simpler.
- **Fargate:** serverless compute for ECS or EKS (no nodes to manage).

**One-liner:** EKS for Kubernetes standards and portability, ECS for AWS-only simplicity, Fargate when you don't want to manage nodes.

## 46. Pod AWS permissions (IRSA / Pod Identity)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/app-role
```
**One-liner:** IRSA maps a Kubernetes service account to an IAM role through the cluster's OIDC provider, so each pod gets least-privilege temporary credentials instead of using the node role. EKS Pod Identity is the newer, simpler alternative.

## 47. CloudWatch: metrics, logs, alarms
```hcl
resource "aws_cloudwatch_metric_alarm" "cpu" {
  alarm_name          = "high-cpu"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  statistic           = "Average"
  period              = 300
  evaluation_periods  = 2
  threshold           = 80
  comparison_operator = "GreaterThanThreshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```
**One-liner:** Metrics for numbers, Logs for text, Alarms trigger SNS or Auto Scaling actions. Memory and disk need the CloudWatch agent.

## 48. High availability vs disaster recovery
- **HA:** survive a component or AZ failure (multi-AZ, load balancers, ASG).
- **DR:** recover from a region-level failure.
- **RTO:** how long you can be down. **RPO:** how much data you can lose.
- DR patterns: Backup and Restore, Pilot Light, Warm Standby, Multi-Site Active/Active.

**One-liner:** HA is within a region. DR is across regions, and the pattern is chosen by RTO and RPO versus cost.

## 49. Cost optimisation
- Right-size instances and delete idle resources
- Savings Plans or Reserved for steady load, Spot for flexible jobs
- S3 lifecycle and Intelligent-Tiering
- Fewer NAT gateways and cross-AZ traffic, use VPC endpoints
- Budgets, Cost Explorer and tagging for visibility

**One-liner:** Measure with tags and Cost Explorer, then right-size, commit and automate cleanup.

## 50. Highly available three-tier app
```
Route 53 -> CloudFront/WAF -> ALB (public subnets, 2+ AZ)
         -> App tier: ASG (private subnets, 2+ AZ)
         -> DB tier: RDS Multi-AZ (isolated subnets) + read replicas
```
- NAT Gateway per AZ for outbound access
- Security groups chained: ALB SG -> App SG -> DB SG
- S3 for static content, Secrets Manager for credentials, CloudWatch for monitoring

**One-liner:** Spread every tier across at least two AZs, isolate tiers with subnets and chained security groups, and keep the database in the most private subnet.

---

# Scenario Questions

## S1. Someone changed a resource manually in the console
```bash
terraform plan -refresh-only      # confirm the drift
terraform apply                   # revert it, or update the code to match and apply
```
**Answer:** Detect with a plan, decide whether the manual change is wanted, then revert it or codify it. Prevent it with restricted console access and scheduled drift checks.

## S2. `terraform apply` is stuck on a state lock
```bash
terraform force-unlock <LOCK_ID>
```
**Answer:** First confirm no other run is active (check CI and teammates), because force-unlocking during a live apply can corrupt state.

## S3. Rename a resource without recreating it
```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}
```
**Answer:** Use a `moved` block, or `terraform state mv` on older versions. Plan should show no destroy.

## S4. EC2 in a private subnet can't reach the internet
Check in order:
1. Private route table has `0.0.0.0/0 -> NAT Gateway`
2. The NAT Gateway is in a **public** subnet, with an Elastic IP
3. The public subnet's route table has `0.0.0.0/0 -> IGW`
4. Instance SG egress and subnet NACL (both directions, including ephemeral ports) allow the traffic
5. DNS resolution is enabled in the VPC

**Answer:** Walk the path: instance SG, NACL, route table, NAT, IGW.

## S5. App on EC2 can't read a secret
Check in order:
1. An instance profile (IAM role) is attached to the instance
2. The role allows `secretsmanager:GetSecretValue` on the secret's ARN
3. If it uses a customer-managed KMS key: `kms:Decrypt` on the role and the key policy
4. Network path to Secrets Manager (NAT or the `secretsmanager` VPC endpoint)
5. Correct region and secret name, and no resource policy on the secret blocking access

**Answer:** Role, then policy, then KMS, then network, then the secret's resource policy.
