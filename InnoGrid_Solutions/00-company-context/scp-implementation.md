# OU Service Boundary SCPs — Implementation Guide

## Overview

This guide implements 5 OU-specific Service Control Policies that enforce the **Key Services** column defined in `aws-account-structure.md`. These SCPs ensure each AWS OU only runs services consistent with its purpose, preventing service sprawl and reducing attack surface.

### Prerequisites

- AWS Organizations configured with all 5 OUs and accounts created
- AWS CLI configured with Organizations admin permissions (Management account)
- Terraform 1.5+ if using infrastructure-as-code approach
- `iam:PassRole` for the Terraform execution role to call `organizations:CreatePolicy` and `organizations:AttachPolicy`

### SCP Summary

| SCP | Target OU | Strategy |
|-----|-----------|----------|
| `Deny-NonManagementServices` | Management OU (`ou-xxxx-mgmt`) | Allow-list: only org admin, identity, billing, logging |
| `Deny-NonSecurityServices` | Security OU (`ou-xxxx-security`) | Allow-list: only security monitoring and audit |
| `Deny-SecurityToolTampering` | Infrastructure OU (`ou-xxxx-infra`) | Deny-list: block security tool disablement |
| `Deny-ProductionResourcesInNonprod` | Nonproduction OU (`ou-xxxx-nonprod`) | Deny-list: block prod instance types / RDS classes |
| `Deny-PersistentResourcesInSandbox` | Sandbox OU (`ou-xxxx-sandbox`) | Deny-list: block RDS, Secrets Manager, persistent storage |

---

## Method 1: Terraform

### Directory Structure

```
terraform/
└── scp-boundaries/
    ├── main.tf
    ├── policies/
    │   ├── deny-non-management-services.json
    │   ├── deny-non-security-services.json
    │   ├── deny-security-tool-tampering.json
    │   ├── deny-production-resources-nonprod.json
    │   └── deny-persistent-resources-sandbox.json
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
  region = "us-east-1"
  # Must be run from the Organizations management account
  assume_role {
    role_arn = "arn:aws:iam::111122223333:role/OrganizationAdmin"
  }
}
```

### Data Sources for OUs (`main.tf` continued)

```hcl
data "aws_organizations_organization" "root" {}

locals {
  root_id = data.aws_organizations_organization.root.roots[0].id

  ou_ids = {
    for ou in data.aws_organizations_organization.root.organizational_units :
    ou.name => ou.id
  }
}

# Verify all expected OUs exist — fails plan if misnamed
locals {
  expected_ous = ["Management", "Security", "Infrastructure", "Nonproduction", "Sandbox"]
  ou_check     = [for ou in local.expected_ous : assert(
    contains(keys(local.ou_ids), ou),
    "OU '${ou}' not found in AWS Organizations"
  )]
}
```

### SCP Resources (`main.tf` continued)

```hcl
# ──────────────────────────────────────────────
# SCP 7: Deny-NonManagementServices
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_non_management" {
  name        = "Deny-NonManagementServices"
  description = "Restrict Management OU to org admin, identity, billing, and logging services only"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-non-management-services.json")
}

resource "aws_organizations_policy_attachment" "deny_non_management_attachment" {
  policy_id = aws_organizations_policy.deny_non_management.id
  target_id = local.ou_ids["Management"]
}

# ──────────────────────────────────────────────
# SCP 8: Deny-NonSecurityServices
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_non_security" {
  name        = "Deny-NonSecurityServices"
  description = "Restrict Security OU to security monitoring, detection, and remediation services only"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-non-security-services.json")
}

resource "aws_organizations_policy_attachment" "deny_non_security_attachment" {
  policy_id = aws_organizations_policy.deny_non_security.id
  target_id = local.ou_ids["Security"]
}

# ──────────────────────────────────────────────
# SCP 9: Deny-SecurityToolTampering
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_security_tampering" {
  name        = "Deny-SecurityToolTampering"
  description = "Prevent workload accounts from disabling or reconfiguring centrally-managed security tools"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-security-tool-tampering.json")
}

resource "aws_organizations_policy_attachment" "deny_security_tampering_attachment" {
  policy_id = aws_organizations_policy.deny_security_tampering.id
  target_id = local.ou_ids["Infrastructure"]
}

# ──────────────────────────────────────────────
# SCP 10: Deny-ProductionResourcesInNonprod
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_production_nonprod" {
  name        = "Deny-ProductionResourcesInNonprod"
  description = "Block production-grade instance types and RDS classes in non-production accounts"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-production-resources-nonprod.json")
}

resource "aws_organizations_policy_attachment" "deny_production_nonprod_attachment" {
  policy_id = aws_organizations_policy.deny_production_nonprod.id
  target_id = local.ou_ids["Nonproduction"]
}

# ──────────────────────────────────────────────
# SCP 11: Deny-PersistentResourcesInSandbox
# ──────────────────────────────────────────────
resource "aws_organizations_policy" "deny_persistent_sandbox" {
  name        = "Deny-PersistentResourcesInSandbox"
  description = "Block RDS, Secrets Manager creation, and persistent storage in sandbox accounts"
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/deny-persistent-resources-sandbox.json")
}

resource "aws_organizations_policy_attachment" "deny_persistent_sandbox_attachment" {
  policy_id = aws_organizations_policy.deny_persistent_sandbox.id
  target_id = local.ou_ids["Sandbox"]
}
```

### Outputs (`outputs.tf`)

```hcl
output "scp_policy_ids" {
  value = {
    deny_non_management         = aws_organizations_policy.deny_non_management.id
    deny_non_security           = aws_organizations_policy.deny_non_security.id
    deny_security_tampering     = aws_organizations_policy.deny_security_tampering.id
    deny_production_nonprod     = aws_organizations_policy.deny_production_nonprod.id
    deny_persistent_sandbox     = aws_organizations_policy.deny_persistent_sandbox.id
  }
  description = "IDs of all created SCPs"
}

output "attachments" {
  value = {
    for attachment in [
      aws_organizations_policy_attachment.deny_non_management_attachment,
      aws_organizations_policy_attachment.deny_non_security_attachment,
      aws_organizations_policy_attachment.deny_security_tampering_attachment,
      aws_organizations_policy_attachment.deny_production_nonprod_attachment,
      aws_organizations_policy_attachment.deny_persistent_sandbox_attachment,
    ] : attachment.policy_id => attachment.target_id
  }
  description = "Map of SCP policy IDs to target OU IDs"
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
    Purpose     = "ou-service-boundary-scps"
  }
}
```

### Policy JSON Files

Create each JSON file under `terraform/scp-boundaries/policies/` using the policy documents from `aws-account-structure.md`.

### Deployment

```powershell
# From the terraform/scp-boundaries directory
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

# List OUs and their IDs
aws organizations list-organizational-units-for-parent `
  --parent-id (aws organizations list-roots --query "Roots[0].Id" --output text)

# Store OU IDs in variables (replace with your actual OU IDs)
$MGMT_OU = "ou-xxxx-mgmt"
$SEC_OU = "ou-xxxx-security"
$INFRA_OU = "ou-xxxx-infra"
$NONPROD_OU = "ou-xxxx-nonprod"
$SANDBOX_OU = "ou-xxxx-sandbox"
```

### Step 1: Create Policy Files

Save each JSON policy from `aws-account-structure.md` as a separate file:
- `deny-non-management-services.json`
- `deny-non-security-services.json`
- `deny-security-tool-tampering.json`
- `deny-production-resources-nonprod.json`
- `deny-persistent-resources-sandbox.json`

### Step 2: Create SCPs

```powershell
# Create Deny-NonManagementServices
$Policy1 = aws organizations create-policy `
  --name "Deny-NonManagementServices" `
  --description "Restrict Management OU to org admin, identity, billing, and logging services only" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-non-management-services.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-NonSecurityServices
$Policy2 = aws organizations create-policy `
  --name "Deny-NonSecurityServices" `
  --description "Restrict Security OU to security monitoring, detection, and remediation services only" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-non-security-services.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-SecurityToolTampering
$Policy3 = aws organizations create-policy `
  --name "Deny-SecurityToolTampering" `
  --description "Prevent workload accounts from disabling centrally-managed security tools" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-security-tool-tampering.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-ProductionResourcesInNonprod
$Policy4 = aws organizations create-policy `
  --name "Deny-ProductionResourcesInNonprod" `
  --description "Block production-grade instance types and RDS classes in non-prod accounts" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-production-resources-nonprod.json) `
  --query "Policy.PolicySummary.Id" --output text

# Create Deny-PersistentResourcesInSandbox
$Policy5 = aws organizations create-policy `
  --name "Deny-PersistentResourcesInSandbox" `
  --description "Block RDS, Secrets Manager creation, and persistent storage in sandbox" `
  --type SERVICE_CONTROL_POLICY `
  --content (Get-Content -Raw .\deny-persistent-resources-sandbox.json) `
  --query "Policy.PolicySummary.Id" --output text
```

### Step 3: Attach SCPs to OUs

```powershell
aws organizations attach-policy --policy-id $Policy1 --target-id $MGMT_OU
aws organizations attach-policy --policy-id $Policy2 --target-id $SEC_OU
aws organizations attach-policy --policy-id $Policy3 --target-id $INFRA_OU
aws organizations attach-policy --policy-id $Policy4 --target-id $NONPROD_OU
aws organizations attach-policy --policy-id $Policy5 --target-id $SANDBOX_OU
```

---

## Method 3: AWS Console

### Step 1: Navigate to Organizations

1. Sign in to **Management Account** as Organizations admin
2. Navigate to **AWS Organizations** → **Policies**
3. Select **Service control policies (SCPs)** tab
4. Click **Enable service control policies** if not already enabled

### Step 2: Create Each Policy

For each of the 5 SCPs:

1. Click **Create policy**
2. Enter the **Policy name** from the table above
3. Enter a **Description** (see table)
4. Paste the JSON policy document from `aws-account-structure.md`
5. Click **Create policy**

### Step 3: Attach to OUs

1. In the SCP list, select a policy (e.g., `Deny-NonManagementServices`)
2. Click **Attach**
3. Check the corresponding OU (e.g., **Management**)
4. Click **Attach policy**
5. Repeat for all 5 policies

---

## Verification

### Verify SCPs Are Created

```powershell
# List all SCPs in the organization
aws organizations list-policies --filter SERVICE_CONTROL_POLICY `
  --query "Policies[*].[Name,Id]" --output table

# Expected: 5 new SCPs + any existing global SCPs
```

### Verify SCP Attachments

```powershell
# Check which policies are attached to each OU
aws organizations list-policies-for-target --target-id $MGMT_OU `
  --filter SERVICE_CONTROL_POLICY --query "Policies[*].Name"
# Expected: Contains "Deny-NonManagementServices"

aws organizations list-policies-for-target --target-id $SEC_OU `
  --filter SERVICE_CONTROL_POLICY --query "Policies[*].Name"
# Expected: Contains "Deny-NonSecurityServices"

aws organizations list-policies-for-target --target-id $INFRA_OU `
  --filter SERVICE_CONTROL_POLICY --query "Policies[*].Name"
# Expected: Contains "Deny-SecurityToolTampering"

aws organizations list-policies-for-target --target-id $NONPROD_OU `
  --filter SERVICE_CONTROL_POLICY --query "Policies[*].Name"
# Expected: Contains "Deny-ProductionResourcesInNonprod"

aws organizations list-policies-for-target --target-id $SANDBOX_OU `
  --filter SERVICE_CONTROL_POLICY --query "Policies[*].Name"
# Expected: Contains "Deny-PersistentResourcesInSandbox"
```

### Verify SCP Content (Dry-Run)

```powershell
# Test that a denied action is blocked in the Nonproduction OU
# (requires credentials in the Nonproduction account)
aws ec2 run-instances `
  --image-id ami-0abcdef1234567890 `
  --instance-type p4d.24xlarge `
  --dry-run
# Expected: "An error occurred (UnauthorizedOperation) when calling the RunInstances operation:
#  You are not authorized to perform this operation. Encoded authorization failure message"

# Decode the failure message to confirm SCP denied it
aws sts decode-authorization-message `
  --encoded-message (aws ec2 run-instances ... 2>&1 | Select-String "encoded authorization failure message" | ForEach-Object { $_.ToString().Split(": ")[-1] })
# Check the "FailedDueToSCP" field in the decoded output
```

### Verify Management OU Restriction

```powershell
# Try to launch an EC2 instance in the Management account (should be blocked)
aws ec2 run-instances --image-id ami-0abcdef1234567890 --instance-type t3.micro --dry-run
# Expected: Access denied (Management OU SCP blocks all EC2 actions)
```

### Verify Sandbox Restrictions

```powershell
# Try to create an RDS instance in the Sandbox account
aws rds create-db-instance `
  --db-instance-identifier test-sandbox-block `
  --db-instance-class db.t3.micro `
  --engine mysql `
  --master-username admin `
  --master-user-password TempPassword123 `
  --allocated-storage 20
# Expected: Access denied by SCP
```

---

## Runtime Behavior

### What Happens When an SCP Blocks an Action

```
User (IAM Identity Center)
  │
  ▼
Permission Set (e.g., PowerUserAccess)
  │
  ▼
IAM Policy Evaluation [Allow]
  │
  ▼
SCP Evaluation [Explicit Deny]  ← Blocked here
  │
  ▼
CloudTrail Event: "SCP denied"
  ├─ EventType: AwsApiCall
  ├─ ErrorCode: AccessDenied
  ├─ ErrorMessage: "You are not authorized to perform this operation.
  │   Encoded authorization failure message is available."
  └─ AdditionalEventData.FailedDueToSCP: "Deny-ProductionResourcesInNonprod"
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

### GuardDuty Finding for SCP Violations (via Security Hub)

Blocked actions do not generate GuardDuty findings by default. To monitor:

1. Create a CloudWatch metric filter on `ErrorCode = "AccessDenied" AND FailedDueToSCP` in the CloudTrail log group
2. Set a CloudWatch alarm that publishes to the Security account SNS topic
3. The Security team reviews the alarm and opens a ticket with the account owner

---

## Compliance Mapping

| Control Requirement | SCP | Evidence |
|---|---|---|
| Management OU only runs admin services | `Deny-NonManagementServices` | Attempts to launch EC2/RDS are denied |
| Security OU only runs security tools | `Deny-NonSecurityServices` | No compute or data services can start |
| Workload accounts cannot disable security | `Deny-SecurityToolTampering` | GuardDuty, Security Hub, Config, CloudTrail are protected |
| Nonprod accounts use cost-effective resources | `Deny-ProductionResourcesInNonprod` | Prod instance types and RDS classes are blocked |
| Sandbox environments are ephemeral | `Deny-PersistentResourcesInSandbox` | RDS, Secrets Manager, persistent volumes denied |
| Security tool admin is centralized | SCPs 7 + 9 combine | Only Security account can manage security tools; workload accounts are blocked |

---

## Maintenance

### Adding a New Service to an Allow-List SCP

1. Update the JSON in `aws-account-structure.md` (documentation source of truth)
2. Update the corresponding JSON file in `terraform/scp-boundaries/policies/`
3. Run `terraform apply` or use CLI `aws organizations update-policy --content file://new-policy.json`

### Adding a New OU

When a new OU is created:
1. Determine the Key Services for the OU
2. Create a new SCP following the pattern above
3. Attach to the new OU
4. Document in `aws-account-structure.md`

### Dry-Run Before Applying Changes

```powershell
aws organizations update-policy --policy-id $PolicyId `
  --content (Get-Content -Raw .\updated-policy.json) `
  --dry-run
# Note: --dry-run is not supported by all AWS Organizations APIs.
# Instead, create a new policy version and test before setting it as default:
aws organizations create-policy-version --policy-id $PolicyId `
  --content (Get-Content -Raw .\new-policy.json) `
  --set-as-default
```

### Rollback

```powershell
# Detach a policy
aws organizations detach-policy --policy-id $PolicyId --target-id $OU_ID

# Delete a policy (must be detached first)
aws organizations delete-policy --policy-id $PolicyId
```
