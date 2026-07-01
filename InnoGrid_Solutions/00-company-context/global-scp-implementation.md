# Global SCPs — Implementation Guide

## Overview

This guide implements the 5 Global Service Control Policies defined in `aws-account-structure.md` (Section 3.1, policies 1–5). Unlike the OU-scoped SCPs in `scp-implementation.md`, these policies are attached to the **Root** of the AWS Organizations tree and apply to **every account**, including the management account.

These policies enforce baseline governance controls — root user lockdown, region restriction, approved instance types, encryption requirements, and public S3 prevention — across the entire organization.

### Prerequisites

- AWS Organizations configured with all features enabled
- SCPs enabled in the organization
- AWS CLI configured with Organizations admin permissions (Management account)
- Terraform 1.5+ if using infrastructure-as-code approach

### SCP Summary

| # | SCP Name | Target | Strategy | Description |
|---|----------|--------|----------|-------------|
| 1 | `Deny-RootActivity` | Root | Block all root user actions | Deny root principal except support, billing, and account management |
| 2 | `Deny-NonCompliantRegions` | Root | Region allow-list | Only `eu-west-2`, `eu-west-1` permitted; global services exempted |
| 3 | `Deny-UnapprovedInstanceTypes` | Root | Instance type allow-list | Only `t3.*`, `m5.*`, `c5.*`, `r5.*` families permitted |
| 4 | `Require-Encryption` | Root | Enforce encryption at create time | Deny unencrypted EBS volumes and RDS instances |
| 5 | `Deny-PublicS3Buckets` | Root | Block public S3 configurations | Deny public ACLs, public bucket policies, removal of public access blocks |

---

## Policy Definitions

### SCP 1 — Deny-RootActivity

**Purpose:** Prevent operational use of root user credentials in all accounts. Root access is only permitted for break-glass IAM users/roles — not the root principal itself.

**Exemptions:** The following minimal actions remain available to root for account administration:
- `aws-portal:*` — Billing and cost management console access
- `support:*` — AWS Support case creation
- `account:*` — Account settings and contact information
- `s3:CreateBucket` — Required for CloudTrail log bucket creation from root

**Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootActivity",
      "Effect": "Deny",
      "NotAction": [
        "aws-portal:*",
        "support:*",
        "account:*",
        "s3:CreateBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

**Note:** This SCP targets the `arn:aws:iam::*:root` principal, not IAM users or roles. Break-glass IAM users and JIT elevation roles are unaffected.

---

### SCP 2 — Deny-NonCompliantRegions

**Purpose:** Restrict all AWS API calls to the two approved UK compliance regions (`eu-west-2` London, `eu-west-1` Ireland). API calls to any other region are denied.

**Exemptions:** Global services that operate in `us-east-1` or have no regional endpoint must be excluded to avoid breaking core functionality (IAM, Organizations, Route53, CloudFront, S3, WAF, Support, etc.).

**Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonCompliantRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "acm:*",
        "aws-marketplace:*",
        "aws-portal:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "cur:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ec2:DescribeAccountAttributes",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "mobileanalytics:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:*",
        "sdb:*",
        "servicecatalog:*",
        "shield:*",
        "sns:List*",
        "sns:Get*",
        "support:*",
        "trustedadvisor:*",
        "waf:*",
        "waf-regional:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["eu-west-2", "eu-west-1"]
        }
      }
    }
  ]
}
```

**Note:** When new global services are added by AWS, they may need to be added to the `NotAction` list. Monitor CloudTrail for `AccessDenied` errors with `FailedDueToSCP` after new service launches.

---

### SCP 3 — Deny-UnapprovedInstanceTypes

**Purpose:** Enforce the approved compute instance family list across all accounts. Only cost-optimised, general-purpose families are allowed.

**Approved families:** `t3.*`, `t3a.*`, `t3n.*`, `m5.*`, `m5a.*`, `m5n.*`, `c5.*`, `c5a.*`, `c5n.*`, `r5.*`, `r5a.*`, `r5n.*`

**Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedInstanceTypes",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotLike": {
          "ec2:InstanceType": [
            "t3.*",
            "t3a.*",
            "t3n.*",
            "m5.*",
            "m5a.*",
            "m5n.*",
            "c5.*",
            "c5a.*",
            "c5n.*",
            "r5.*",
            "r5a.*",
            "r5n.*"
          ]
        }
      }
    }
  ]
}
```

**Note:** This only blocks `RunInstances`. Existing running instances are not affected. The `ec2:InstanceType` condition key supports wildcard matching, so `t3.*` covers all `t3` sizes (`t3.nano` through `t3.2xlarge`).

---

### SCP 4 — Require-Encryption

**Purpose:** Enforce encryption at rest for EBS volumes and RDS instances at creation time. Resources created without encryption are denied.

**Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedEBS",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:volume/*",
      "Condition": {
        "BoolIfExists": {
          "ec2:Encrypted": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedRDS",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance",
        "rds:ModifyDBInstance"
      ],
      "Resource": "arn:aws:rds:*:*:db:*",
      "Condition": {
        "BoolIfExists": {
          "rds:StorageEncrypted": "false"
        }
      }
    }
  ]
}
```

**Note:** S3 object encryption is not covered by this SCP. It is enforced via S3 bucket policies and AWS Config rules with auto-remediation (see `security-config.md`). The `BoolIfExists` operator means the condition only applies when the encryption parameter is explicitly set to `false`; if the parameter is omitted and the account has a default encryption setting, the SCP does not fire.

---

### SCP 5 — Deny-PublicS3Buckets

**Purpose:** Prevent any S3 bucket or object from being made publicly accessible. This covers ACL-based access (`public-read`, `public-read-write`) and bucket policies that grant public access.

**Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicS3Acl",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketAcl",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:*:*:*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": ["public-read", "public-read-write"]
        }
      }
    },
    {
      "Sid": "DenyPublicBucketPolicy",
      "Effect": "Deny",
      "Action": "s3:PutBucketPolicy",
      "Resource": "arn:aws:s3:*:*:*",
      "Condition": {
        "StringEquals": {
          "s3:PublicAccessBlockConfiguration.RestrictPublicBuckets": "false"
        }
      }
    }
  ]
}
```

**Note:** This SCP works alongside the AWS account-level S3 Public Access Block setting. Together they provide defence in depth — the account setting blocks public access at the account level, while this SCP prevents any IAM principal from explicitly granting public access via ACLs or bucket policies.

---

## Method 1: Terraform

### Directory Structure

```
terraform/
└── global-scps/
    ├── main.tf
    ├── policies/
    │   ├── deny-root-activity.json
    │   ├── deny-non-compliant-regions.json
    │   ├── deny-unapproved-instance-types.json
    │   ├── require-encryption.json
    │   └── deny-public-s3-buckets.json
    ├── outputs.tf
    └── variables.tf
```

### Provider Configuration (`main.tf`)

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-2"
  # Must be run from the Organizations management account
  assume_role {
    role_arn = "arn:aws:iam::111122223333:role/OrganizationAdmin"
  }
}
```

### Data Sources for Root (`main.tf` continued)

```hcl
data "aws_organizations_organization" "org" {}

locals {
  root_id = data.aws_organizations_organization.org.roots[0].id
}
```

### SCP Resources (`main.tf` continued)

```hcl
# ──────────────────────────────────────────────
# SCP 1: Deny-RootActivity
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_root_activity" {
  name        = "Deny-RootActivity"
  description = "Block root user actions across all accounts; exempt support, billing, and account management"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-root-activity.json")
}

resource "aws_organizations_policy_attachment" "deny_root_activity_attachment" {
  policy_id = aws_organizations_policy.deny_root_activity.id
  target_id = local.root_id
}

# ──────────────────────────────────────────────
# SCP 2: Deny-NonCompliantRegions
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_non_compliant_regions" {
  name        = "Deny-NonCompliantRegions"
  description = "Restrict API calls to eu-west-2 and eu-west-1 only; global services exempted"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-non-compliant-regions.json")
}

resource "aws_organizations_policy_attachment" "deny_non_compliant_regions_attachment" {
  policy_id = aws_organizations_policy.deny_non_compliant_regions.id
  target_id = local.root_id
}

# ──────────────────────────────────────────────
# SCP 3: Deny-UnapprovedInstanceTypes
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_unapproved_instance_types" {
  name        = "Deny-UnapprovedInstanceTypes"
  description = "Allow only t3, m5, c5, r5 instance families across all accounts"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-unapproved-instance-types.json")
}

resource "aws_organizations_policy_attachment" "deny_unapproved_instance_types_attachment" {
  policy_id = aws_organizations_policy.deny_unapproved_instance_types.id
  target_id = local.root_id
}

# ──────────────────────────────────────────────
# SCP 4: Require-Encryption
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "require_encryption" {
  name        = "Require-Encryption"
  description = "Enforce encryption at rest for EBS volumes and RDS instances"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/require-encryption.json")
}

resource "aws_organizations_policy_attachment" "require_encryption_attachment" {
  policy_id = aws_organizations_policy.require_encryption.id
  target_id = local.root_id
}

# ──────────────────────────────────────────────
# SCP 5: Deny-PublicS3Buckets
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_public_s3" {
  name        = "Deny-PublicS3Buckets"
  description = "Block public ACLs and bucket policies across all S3 buckets"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-public-s3-buckets.json")
}

resource "aws_organizations_policy_attachment" "deny_public_s3_attachment" {
  policy_id = aws_organizations_policy.deny_public_s3.id
  target_id = local.root_id
}
```

### Outputs (`outputs.tf`)

```hcl
output "scp_policy_ids" {
  value = {
    deny_root_activity          = aws_organizations_policy.deny_root_activity.id
    deny_non_compliant_regions  = aws_organizations_policy.deny_non_compliant_regions.id
    deny_unapproved_instance_types = aws_organizations_policy.deny_unapproved_instance_types.id
    require_encryption          = aws_organizations_policy.require_encryption.id
    deny_public_s3              = aws_organizations_policy.deny_public_s3.id
  }
  description = "IDs of all created global SCPs"
}

output "attachments" {
  value = {
    for attachment in [
      aws_organizations_policy_attachment.deny_root_activity_attachment,
      aws_organizations_policy_attachment.deny_non_compliant_regions_attachment,
      aws_organizations_policy_attachment.deny_unapproved_instance_types_attachment,
      aws_organizations_policy_attachment.require_encryption_attachment,
      aws_organizations_policy_attachment.deny_public_s3_attachment,
    ] : attachment.policy_id => attachment.target_id
  }
  description = "Map of SCP policy IDs to root target ID"
}
```

### Variables (`variables.tf`)

```hcl
variable "tags" {
  description = "Tags to apply to all SCP resources"
  type        = map(string)
  default = {
    Environment = "production"
    ManagedBy   = "terraform"
    Purpose     = "global-scps"
  }
}
```

### Policy JSON Files

Create each JSON file under `terraform/global-scps/policies/` using the policy documents from the **Policy Definitions** section above:

- `deny-root-activity.json`
- `deny-non-compliant-regions.json`
- `deny-unapproved-instance-types.json`
- `require-encryption.json`
- `deny-public-s3-buckets.json`

### Deployment

```powershell
# From the terraform/global-scps directory
terraform init
terraform plan -out=scp-plan.tfplan
terraform apply scp-plan.tfplan
```

---

## Method 2: AWS CLI

### Prerequisites

```powershell
# Verify Organizations access
aws organizations describe-organization

# Get the Root OU ID
$ROOT_ID = aws organizations list-roots --query "Roots[0].Id" --output text
Write-Host "Root ID: $ROOT_ID"
```

### Step 1: Create Policy Files

Create each JSON file using the policy documents from the **Policy Definitions** section above:

- `deny-root-activity.json`
- `deny-non-compliant-regions.json`
- `deny-unapproved-instance-types.json`
- `require-encryption.json`
- `deny-public-s3-buckets.json`

### Step 2: Create SCPs

```powershell
# Create Deny-RootActivity
$Policy1 = aws organizations create-policy `
  --name "Deny-RootActivity" `
  --description "Block root user actions across all accounts; exempt support, billing, and account management" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-root-activity.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-NonCompliantRegions
$Policy2 = aws organizations create-policy `
  --name "Deny-NonCompliantRegions" `
  --description "Restrict API calls to eu-west-2 and eu-west-1 only; global services exempted" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-non-compliant-regions.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-UnapprovedInstanceTypes
$Policy3 = aws organizations create-policy `
  --name "Deny-UnapprovedInstanceTypes" `
  --description "Allow only t3, m5, c5, r5 instance families across all accounts" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-unapproved-instance-types.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Require-Encryption
$Policy4 = aws organizations create-policy `
  --name "Require-Encryption" `
  --description "Enforce encryption at rest for EBS volumes and RDS instances" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\require-encryption.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-PublicS3Buckets
$Policy5 = aws organizations create-policy `
  --name "Deny-PublicS3Buckets" `
  --description "Block public ACLs and bucket policies across all S3 buckets" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-public-s3-buckets.json) `
  --query "Policy.PolicySummary.Id" --output text
```

### Step 3: Attach SCPs to Root

```powershell
aws organizations attach-policy --policy-id $Policy1 --target-id $ROOT_ID
aws organizations attach-policy --policy-id $Policy2 --target-id $ROOT_ID
aws organizations attach-policy --policy-id $Policy3 --target-id $ROOT_ID
aws organizations attach-policy --policy-id $Policy4 --target-id $ROOT_ID
aws organizations attach-policy --policy-id $Policy5 --target-id $ROOT_ID
```

---

## Method 3: AWS Console

### Step 1: Navigate to Organizations

1. Sign in to **Management Account** as Organizations admin
2. Navigate to **AWS Organizations** → **Policies**
3. Select **Service control policies (SCPs)** tab
4. Confirm **Service control policies** is enabled (toggle if needed)

### Step 2: Create Each Policy

For each of the 5 SCPs:

1. Click **Create policy**
2. Enter the **Policy name** from the SCP Summary table
3. Enter a **Description** (see SCP Summary table)
4. Switch to the **JSON** tab
5. Paste the JSON policy document from the **Policy Definitions** section above
6. Click **Create policy**

### Step 3: Attach to Root

1. In the SCP list, select a policy (e.g., `Deny-RootActivity`)
2. Click **Attach**
3. Select the **Root** organizational unit
4. Click **Attach policy**
5. Repeat for all 5 policies

---

## Verification

### Verify SCPs Are Created

```powershell
# List all SCPs in the organization
aws organizations list-policies --filter SERVICE_CONTROL_POLICY `
  --query "Policies[*].[Name,Id]" --output table

# Expected: 5 new global SCPs + any existing OU-specific SCPs
```

### Verify SCP Attachments to Root

```powershell
# Check which policies are attached to the Root
$ROOT_ID = aws organizations list-roots --query "Roots[0].Id" --output text
aws organizations list-policies-for-target --target-id $ROOT_ID `
  --filter SERVICE_CONTROL_POLICY --query "Policies[*].Name"
# Expected:
#   Deny-RootActivity
#   Deny-NonCompliantRegions
#   Deny-UnapprovedInstanceTypes
#   Require-Encryption
#   Deny-PublicS3Buckets
```

### Verify Root Activity Block

```powershell
# Attempt an action as root principal (e.g., create an IAM user)
# Requires root credentials in any member account
aws iam create-user --user-name test-root-block
# Expected: Access denied (only works if assumed via break-glass role or allowed services)
```

### Verify Region Restriction

```powershell
# Try to describe EC2 in a blocked region (e.g., us-east-1)
# Requires credentials in any member account
aws ec2 describe-instances --region us-east-1 --dry-run
# Expected: Access denied by SCP

# Verify approved regions still work
aws ec2 describe-instances --region eu-west-2 --dry-run
# Expected: DryRun flag set (i.e., request would succeed if --dry-run were removed)
```

### Verify Approved Instance Types

```powershell
# Try to launch a blocked instance type
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type p4d.24xlarge --dry-run
# Expected: Access denied by SCP

# Try to launch an approved instance type
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type t3.micro --dry-run
# Expected: DryRun flag set (if other policies don't block EC2 in that account)
```

### Verify Encryption Enforcement

```powershell
# Try to create an unencrypted RDS instance
aws rds create-db-instance `
  --db-instance-identifier test-encrypt-block `
  --db-instance-class db.t3.micro `
  --engine mysql `
  --master-username admin `
  --master-user-password TempPassword123 `
  --allocated-storage 20 `
  --no-storage-encrypted
# Expected: Access denied by SCP

# Try to launch an EC2 instance without EBS encryption
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type t3.micro `
  --block-device-mappings DeviceName=/dev/sda1,Ebs={VolumeSize=20,Encrypted=false} --dry-run
# Expected: Access denied by SCP
```

### Verify Public S3 Block

```powershell
# Try to set a public-read ACL on a bucket
aws s3api put-bucket-acl --bucket test-block-public --acl public-read
# Expected: Access denied by SCP

# Try to disable public access block on a bucket
aws s3api put-public-access-block --bucket test-block-public `
  --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
# Expected: Access denied by SCP
```

### Decode a Denied Request

```powershell
# When you receive an encoded authorization failure message, decode it:
aws sts decode-authorization-message `
  --encoded-message "<encoded_message>" `
  --query "DecodedMessage" --output text | ConvertFrom-Json | Select-Object -ExpandProperty context
# Check the "FailedDueToSCP" field to confirm which SCP blocked the request
```

---

## Runtime Behaviour

### SCP Evaluation Order

The Global SCPs and OU-specific SCPs are evaluated together as a single policy set. An explicit `Deny` from **any** attached SCP blocks the action:

```
User (IAM Identity Center / IAM User)
  │
  ▼
Permission Set / IAM Policy → [Allow | Deny]
  │
  ▼
Global SCPs (attached to Root) → [Explicit Deny]
  │  └─ Deny-RootActivity
  │  └─ Deny-NonCompliantRegions
  │  └─ Deny-UnapprovedInstanceTypes
  │  └─ Require-Encryption
  │  └─ Deny-PublicS3Buckets
  │
  ▼
OU SCPs (attached to child OU) → [Explicit Deny]
  │  └─ e.g., Deny-NonManagementServices
  │  └─ e.g., Deny-SecurityToolTampering
  │
  ▼
Result: Allowed only if no SCP at any level denies it
```

### CloudTrail Query for SCP Denials

```sql
-- Athena query on CloudTrail logs
SELECT
  eventsource,
  eventname,
  useridentity.arn,
  errorcode,
  errormessage,
  json_extract_scalar(additionaleventdata, '$.failedDueToSCP') as scp_name
FROM cloudtrail_logs
WHERE errorcode = 'AccessDenied'
  AND json_extract_scalar(additionaleventdata, '$.failedDueToSCP') IS NOT NULL
  AND eventtime > date_add('day', -7, now())
ORDER BY eventtime DESC;
```

---

## Compliance Mapping

| Control Requirement | SCP | Evidence |
|---|---|---|
| Root user is not used for day-to-day operations | `Deny-RootActivity` | Root principal actions are denied; CloudTrail shows root ARN in `userIdentity` with `AccessDenied` |
| Workloads run only in UK compliance regions | `Deny-NonCompliantRegions` | API calls to `us-east-1`, `ap-southeast-2`, etc. return `AccessDenied` with `FailedDueToSCP` |
| Only approved instance families are used | `Deny-UnapprovedInstanceTypes` | EC2 `RunInstances` with `g4dn.*`, `p4d.*`, etc. are denied |
| Data at rest is encrypted | `Require-Encryption` | Unencrypted EBS volumes and RDS instances cannot be created |
| S3 buckets are not publicly accessible | `Deny-PublicS3Buckets` | Public ACLs, public bucket policies, and disabling public access blocks are denied |

---

## Maintenance

### Adding a New Approved Instance Type

1. Update the `aws-account-structure.md` documentation
2. Add the new family to the `ec2:InstanceType` allow-list in `deny-unapproved-instance-types.json`
3. Run `terraform apply` or use:
   ```powershell
   aws organizations update-policy --policy-id $PolicyId `
     --content (Get-Content -Raw .\deny-unapproved-instance-types.json)
   ```

### Adding a New Global Service to the Region Exemption List

When AWS launches a new global service, you may see `AccessDenied` errors in CloudTrail:

1. Identify the service prefix from the CloudTrail event (e.g., `newservice:*`)
2. Add it to the `NotAction` list in `deny-non-compliant-regions.json`
3. Apply the change

### Removing an SCP (Rollback)

```powershell
# Detach from Root
aws organizations detach-policy --policy-id $PolicyId --target-id $ROOT_ID

# Delete the policy (must be detached first)
aws organizations delete-policy --policy-id $PolicyId
```

### Testing Policy Changes (Versioning)

```powershell
# Create a new policy version (can have up to 5 versions)
aws organizations create-policy-version --policy-id $PolicyId `
  --content (Get-Content -Raw .\new-policy.json)

# List versions
aws organizations list-policy-versions --policy-id $PolicyId

# Set the new version as default (active)
aws organizations update-policy --policy-id $PolicyId `
  --set-as-default
```
