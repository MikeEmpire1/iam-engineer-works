# Scenario 0: Foundation — IAM Identity Centre Implementation

## Prerequisites

- AWS Organizations enabled with all features
- Management account admin access (AWS CLI + Console)
- IAM Identity Centre enabled in the Management account (us-east-1 or eu-west-2)
- Terraform 1.5+ (if using infrastructure-as-code)
- Entra ID Global Admin access (for SCIM setup)

### Reference Values

| Resource | Value |
|---|---|
| Identity Store ID | `d-9a7b8c6d5e` |
| Instance ARN | `arn:aws:sso:::instance/ssoins-1234567890abcdef` |
| Management Account | `111122223333` (`inno-mgmt`) |
| Security Account | `444455556666` (`inno-security`) |
| Nonproduction Account | `123456789012` (`inno-nonprod`) |
| Primary Region | `eu-west-2` |

---

## 1. Enable IAM Identity Centre

### Method 1: AWS Console

1. Sign in to the **Management Account** as an administrator
2. Navigate to **AWS IAM Identity Centre** (formerly AWS SSO)
3. Click **Enable**
4. Select **Enable with AWS Organizations** (the Identity Centre instance will be created)
5. Choose identity source: **Identity Centre directory** (for now; Entra ID will be added in Section 7)

### Method 2: AWS CLI

```powershell
# IAM Identity Centre is enabled via AWS Organizations.
# First, verify Organizations is ready:
aws organizations describe-organization

# The Identity Centre instance is created by AWS when enabled via Console.
# To check if it's enabled:
aws sso-admin list-instances --query "Instances[*].[InstanceArn, IdentityStoreId]" --output table
```

### Method 3: Terraform

```hcl
# IAM Identity Centre cannot be created via Terraform directly —
# it must be enabled via Console or CLI first.
# Once enabled, Terraform can manage resources within it.

data "aws_ssoadmin_instances" "this" {}

locals {
  identity_store_id = tolist(data.aws_ssoadmin_instances.this.identity_store_ids)[0]
  instance_arn      = tolist(data.aws_ssoadmin_instances.this.arns)[0]
}

output "identity_store_id" {
  value = local.identity_store_id
}

output "instance_arn" {
  value = local.instance_arn
}
```

---

## 2. Create Groups

### Terraform

```hcl
# terraform/identity-centre/groups.tf

locals {
  identity_store_id = "d-9a7b8c6d5e"
}

resource "aws_identitystore_group" "engineering" {
  identity_store_id = local.identity_store_id
  display_name      = "engineering"
  description       = "All engineers — grants baseline DevAccess to Nonproduction"
}

resource "aws_identitystore_group" "platform_engineers" {
  identity_store_id = local.identity_store_id
  display_name      = "platform-engineers"
  description       = "Platform engineering team"
}

resource "aws_identitystore_group" "app_dev" {
  identity_store_id = local.identity_store_id
  display_name      = "app-dev"
  description       = "Application development team"
}

resource "aws_identitystore_group" "qa_engineers" {
  identity_store_id = local.identity_store_id
  display_name      = "qa-engineers"
  description       = "QA and testing team"
}

resource "aws_identitystore_group" "engineering_managers" {
  identity_store_id = local.identity_store_id
  display_name      = "engineering-managers"
  description       = "Engineering management (M2+ level)"
}

resource "aws_identitystore_group" "it_security" {
  identity_store_id = local.identity_store_id
  display_name      = "it-security"
  description       = "IT and Security department"
}

resource "aws_identitystore_group" "iam_team" {
  identity_store_id = local.identity_store_id
  display_name      = "iam-team"
  description       = "IAM administration team — privileged group"
}

resource "aws_identitystore_group" "soc_team" {
  identity_store_id = local.identity_store_id
  display_name      = "soc-team"
  description       = "Security operations team"
}

resource "aws_identitystore_group" "service_desk" {
  identity_store_id = local.identity_store_id
  display_name      = "service-desk"
  description       = "IT support and service desk team"
}

resource "aws_identitystore_group" "break_glass" {
  identity_store_id = local.identity_store_id
  display_name      = "break-glass"
  description       = "Emergency access group — privileged, monitored"
}
```

### AWS CLI

```powershell
# Create each group
$IdentityStoreId = "d-9a7b8c6d5e"

function Create-Group($displayName, $description) {
  aws identitystore create-group `
    --identity-store-id $IdentityStoreId `
    --display-name $displayName `
    --description $description
}

Create-Group "engineering" "All engineers — grants baseline DevAccess to Nonproduction"
Create-Group "platform-engineers" "Platform engineering team"
Create-Group "app-dev" "Application development team"
Create-Group "qa-engineers" "QA and testing team"
Create-Group "engineering-managers" "Engineering management (M2+ level)"
Create-Group "it-security" "IT and Security department"
Create-Group "iam-team" "IAM administration team — privileged group"
Create-Group "soc-team" "Security operations team"
Create-Group "service-desk" "IT support and service desk team"
Create-Group "break-glass" "Emergency access group — privileged, monitored"
```

### AWS Console

1. **AWS Console** → IAM Identity Centre → **Groups** → **Create group**
2. For each group in the table:
   - Enter **Group name** (e.g., `engineering`)
   - Enter **Description** (from the table above)
   - Click **Create group**
3. Repeat for all 10 groups

---

## 3. Create Permission Sets

### Terraform

```hcl
# terraform/identity-centre/permission-sets.tf

locals {
  instance_arn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
}

# ──────────────────────────────────────────────
# DevAccess — for engineering users in Nonproduction
# ──────────────────────────────────────────────
resource "aws_ssoadmin_permission_set" "dev_access" {
  instance_arn     = local.instance_arn
  name             = "DevAccess"
  session_duration = "PT8H"
  relay_state      = "https://console.aws.amazon.com/ec2"
}

resource "aws_ssoadmin_managed_policy_attachment" "dev_access_readonly" {
  instance_arn       = local.instance_arn
  managed_policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
}

resource "aws_ssoadmin_permission_set_inline_policy" "dev_access_inline" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  inline_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket",
          "s3:GetObject"
        ]
        Resource = [
          "arn:aws:s3:::inno-nonprod-dev-artifacts",
          "arn:aws:s3:::inno-nonprod-dev-artifacts/*"
        ]
      }
    ]
  })
}

# ──────────────────────────────────────────────
# ReadOnly — for service desk and general read access
# ──────────────────────────────────────────────
resource "aws_ssoadmin_permission_set" "read_only" {
  instance_arn     = local.instance_arn
  name             = "ReadOnly"
  session_duration = "PT4H"
  relay_state      = "https://console.aws.amazon.com"
}

resource "aws_ssoadmin_managed_policy_attachment" "read_only_attach" {
  instance_arn       = local.instance_arn
  managed_policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
  permission_set_arn = aws_ssoadmin_permission_set.read_only.arn
}

# ──────────────────────────────────────────────
# SecurityAudit — for SOC team
# ──────────────────────────────────────────────
resource "aws_ssoadmin_permission_set" "security_audit" {
  instance_arn     = local.instance_arn
  name             = "SecurityAudit"
  session_duration = "PT4H"
  relay_state      = "https://console.aws.amazon.com/securityhub"
}

resource "aws_ssoadmin_managed_policy_attachment" "security_audit_attach" {
  instance_arn       = local.instance_arn
  managed_policy_arn = "arn:aws:iam::aws:policy/SecurityAudit"
  permission_set_arn = aws_ssoadmin_permission_set.security_audit.arn
}

# ──────────────────────────────────────────────
# AdministratorAccess — for IAM team and break-glass (JIT)
# ──────────────────────────────────────────────
resource "aws_ssoadmin_permission_set" "admin_access" {
  instance_arn     = local.instance_arn
  name             = "AdministratorAccess"
  session_duration = "PT1H"
  relay_state      = "https://console.aws.amazon.com"
}

resource "aws_ssoadmin_managed_policy_attachment" "admin_access_attach" {
  instance_arn       = local.instance_arn
  managed_policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  permission_set_arn = aws_ssoadmin_permission_set.admin_access.arn
}

# ──────────────────────────────────────────────
# PowerUserAccess — for engineers who need more than read-only
# ──────────────────────────────────────────────
resource "aws_ssoadmin_permission_set" "power_user_access" {
  instance_arn     = local.instance_arn
  name             = "PowerUserAccess"
  session_duration = "PT4H"
  relay_state      = "https://console.aws.amazon.com"
}

resource "aws_ssoadmin_managed_policy_attachment" "power_user_attach" {
  instance_arn       = local.instance_arn
  managed_policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
  permission_set_arn = aws_ssoadmin_permission_set.power_user_access.arn
}
```

### AWS CLI

```powershell
$InstanceArn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"

# Create DevAccess permission set
$DevAccessArn = aws sso-admin create-permission-set `
  --instance-arn $InstanceArn `
  --name "DevAccess" `
  --session-duration "PT8H" `
  --relay-state "https://console.aws.amazon.com/ec2" `
  --query "PermissionSet.PermissionSetArn" --output text

# Attach ReadOnlyAccess managed policy
aws sso-admin attach-managed-policy-to-permission-set `
  --instance-arn $InstanceArn `
  --permission-set-arn $DevAccessArn `
  --managed-policy-arn "arn:aws:iam::aws:policy/ReadOnlyAccess"

# Create inline policy for S3 dev artifacts
aws sso-admin put-inline-policy-to-permission-set `
  --instance-arn $InstanceArn `
  --permission-set-arn $DevAccessArn `
  --inline-policy (Get-Content -Raw .\policies\dev-access-inline.json)

# Repeat for ReadOnly, SecurityAudit, AdministratorAccess, PowerUserAccess
# (omitted for brevity — same pattern with different names/policies)
```

### AWS Console

1. **AWS Console** → IAM Identity Centre → **Permission sets** → **Create permission set**
2. Select **Custom permission set**
3. Enter **Name** (e.g., `DevAccess`)
4. Set **Session duration** (e.g., `8 hours`)
5. Set **Relay state** (e.g., `https://console.aws.amazon.com/ec2`)
6. Under **Managed policies**, select `ReadOnlyAccess`
7. Under **Inline policy**, paste the S3 inline policy JSON
8. Click **Create**
9. Repeat for: `ReadOnly`, `SecurityAudit`, `AdministratorAccess` (1h session), `PowerUserAccess`

---

## 4. Provision Users

### Terraform

```hcl
# terraform/identity-centre/users.tf

locals {
  identity_store_id = "d-9a7b8c6d5e"
}

# ── Engineering ──

resource "aws_identitystore_user" "james_okafor" {
  identity_store_id = local.identity_store_id
  display_name      = "James Okafor"
  user_name         = "james.okafor@innogrid.com"
  user_type         = "Exec"

  name {
    given_name  = "James"
    family_name = "Okafor"
  }

  emails {
    value   = "james.okafor@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "priya_sharma" {
  identity_store_id = local.identity_store_id
  display_name      = "Priya Sharma"
  user_name         = "priya.sharma@innogrid.com"
  user_type         = "M2"

  name {
    given_name  = "Priya"
    family_name = "Sharma"
  }

  emails {
    value   = "priya.sharma@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "derek_jones" {
  identity_store_id = local.identity_store_id
  display_name      = "Derek Jones"
  user_name         = "derek.jones@innogrid.com"
  user_type         = "M2"

  name {
    given_name  = "Derek"
    family_name = "Jones"
  }

  emails {
    value   = "derek.jones@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "lisa_kim" {
  identity_store_id = local.identity_store_id
  display_name      = "Lisa Kim"
  user_name         = "lisa.kim@innogrid.com"
  user_type         = "M2"

  name {
    given_name  = "Lisa"
    family_name = "Kim"
  }

  emails {
    value   = "lisa.kim@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "alex_rivera" {
  identity_store_id = local.identity_store_id
  display_name      = "Alex Rivera"
  user_name         = "alex.rivera@innogrid.com"
  user_type         = "IC4"

  name {
    given_name  = "Alex"
    family_name = "Rivera"
  }

  emails {
    value   = "alex.rivera@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "sam_green" {
  identity_store_id = local.identity_store_id
  display_name      = "Sam Green"
  user_name         = "sam.green@innogrid.com"
  user_type         = "IC3"

  name {
    given_name  = "Sam"
    family_name = "Green"
  }

  emails {
    value   = "sam.green@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "maya_johnson" {
  identity_store_id = local.identity_store_id
  display_name      = "Maya Johnson"
  user_name         = "maya.johnson@innogrid.com"
  user_type         = "IC3"

  name {
    given_name  = "Maya"
    family_name = "Johnson"
  }

  emails {
    value   = "maya.johnson@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "ethan_brown" {
  identity_store_id = local.identity_store_id
  display_name      = "Ethan Brown"
  user_name         = "ethan.brown@innogrid.com"
  user_type         = "IC2"

  name {
    given_name  = "Ethan"
    family_name = "Brown"
  }

  emails {
    value   = "ethan.brown@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "chloe_wilson" {
  identity_store_id = local.identity_store_id
  display_name      = "Chloe Wilson"
  user_name         = "chloe.wilson@innogrid.com"
  user_type         = "IC2"

  name {
    given_name  = "Chloe"
    family_name = "Wilson"
  }

  emails {
    value   = "chloe.wilson@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

# ── IT & Security ──

resource "aws_identitystore_user" "ryan_mitchell" {
  identity_store_id = local.identity_store_id
  display_name      = "Ryan Mitchell"
  user_name         = "ryan.mitchell@innogrid.com"
  user_type         = "M2"

  name {
    given_name  = "Ryan"
    family_name = "Mitchell"
  }

  emails {
    value   = "ryan.mitchell@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "aisha_patel" {
  identity_store_id = local.identity_store_id
  display_name      = "Aisha Patel"
  user_name         = "aisha.patel@innogrid.com"
  user_type         = "IC4"

  name {
    given_name  = "Aisha"
    family_name = "Patel"
  }

  emails {
    value   = "aisha.patel@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "miguel_torres" {
  identity_store_id = local.identity_store_id
  display_name      = "Miguel Torres"
  user_name         = "miguel.torres@innogrid.com"
  user_type         = "IC3"

  name {
    given_name  = "Miguel"
    family_name = "Torres"
  }

  emails {
    value   = "miguel.torres@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "tanya_brooks" {
  identity_store_id = local.identity_store_id
  display_name      = "Tanya Brooks"
  user_name         = "tanya.brooks@innogrid.com"
  user_type         = "M2"

  name {
    given_name  = "Tanya"
    family_name = "Brooks"
  }

  emails {
    value   = "tanya.brooks@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "jake_hoffman" {
  identity_store_id = local.identity_store_id
  display_name      = "Jake Hoffman"
  user_name         = "jake.hoffman@innogrid.com"
  user_type         = "IC4"

  name {
    given_name  = "Jake"
    family_name = "Hoffman"
  }

  emails {
    value   = "jake.hoffman@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "olivia_reed" {
  identity_store_id = local.identity_store_id
  display_name      = "Olivia Reed"
  user_name         = "olivia.reed@innogrid.com"
  user_type         = "IC2"

  name {
    given_name  = "Olivia"
    family_name = "Reed"
  }

  emails {
    value   = "olivia.reed@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "carlos_mendez" {
  identity_store_id = local.identity_store_id
  display_name      = "Carlos Mendez"
  user_name         = "carlos.mendez@innogrid.com"
  user_type         = "M1"

  name {
    given_name  = "Carlos"
    family_name = "Mendez"
  }

  emails {
    value   = "carlos.mendez@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "emily_zhao" {
  identity_store_id = local.identity_store_id
  display_name      = "Emily Zhao"
  user_name         = "emily.zhao@innogrid.com"
  user_type         = "IC3"

  name {
    given_name  = "Emily"
    family_name = "Zhao"
  }

  emails {
    value   = "emily.zhao@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}

resource "aws_identitystore_user" "kevin_nguyen" {
  identity_store_id = local.identity_store_id
  display_name      = "Kevin Nguyen"
  user_name         = "kevin.nguyen@innogrid.com"
  user_type         = "IC1"

  name {
    given_name  = "Kevin"
    family_name = "Nguyen"
  }

  emails {
    value   = "kevin.nguyen@innogrid.com"
    primary = true
  }

  locale = "en-GB"
}
```

### AWS CLI

```powershell
$IdentityStoreId = "d-9a7b8c6d5e"

function Create-User($givenName, $familyName, $email, $userType) {
  aws identitystore create-user `
    --identity-store-id $IdentityStoreId `
    --user-name $email `
    --display-name "$givenName $familyName" `
    --name GivenName=$givenName,FamilyName=$familyName `
    --emails "[{\"value\":\"$email\",\"primary\":true}]" `
    --user-type $userType `
    --locale "en-GB"
}

Create-User "James" "Okafor" "james.okafor@innogrid.com" "Exec"
Create-User "Priya" "Sharma" "priya.sharma@innogrid.com" "M2"
Create-User "Derek" "Jones" "derek.jones@innogrid.com" "M2"
Create-User "Lisa" "Kim" "lisa.kim@innogrid.com" "M2"
Create-User "Alex" "Rivera" "alex.rivera@innogrid.com" "IC4"
Create-User "Sam" "Green" "sam.green@innogrid.com" "IC3"
Create-User "Maya" "Johnson" "maya.johnson@innogrid.com" "IC3"
Create-User "Ethan" "Brown" "ethan.brown@innogrid.com" "IC2"
Create-User "Chloe" "Wilson" "chloe.wilson@innogrid.com" "IC2"
Create-User "Ryan" "Mitchell" "ryan.mitchell@innogrid.com" "M2"
Create-User "Aisha" "Patel" "aisha.patel@innogrid.com" "IC4"
Create-User "Miguel" "Torres" "miguel.torres@innogrid.com" "IC3"
Create-User "Tanya" "Brooks" "tanya.brooks@innogrid.com" "M2"
Create-User "Jake" "Hoffman" "jake.hoffman@innogrid.com" "IC4"
Create-User "Olivia" "Reed" "olivia.reed@innogrid.com" "IC2"
Create-User "Carlos" "Mendez" "carlos.mendez@innogrid.com" "M1"
Create-User "Emily" "Zhao" "emily.zhao@innogrid.com" "IC3"
Create-User "Kevin" "Nguyen" "kevin.nguyen@innogrid.com" "IC1"
```

### AWS Console

1. **AWS Console** → IAM Identity Centre → **Users** → **Add user**
2. For each user from the provisioning matrix:
   - Enter **Username** (email)
   - Enter **Email address**
   - Enter **First name**, **Last name**
   - Enter **Display name**
   - Click **Next** (skip group assignment for now — handled in Section 5)
   - Review and **Create user**
3. Repeat for all 18 active users

---

## 5. Assign Group Memberships

### Terraform

```hcl
# terraform/identity-centre/group-memberships.tf

locals {
  identity_store_id = "d-9a7b8c6d5e"
}

# ── Engineering Managers ──

resource "aws_identitystore_group_membership" "james_okafor_engineering_managers" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering_managers.group_id
  member_id         = aws_identitystore_user.james_okafor.user_id
}

resource "aws_identitystore_group_membership" "priya_sharma_engineering_managers" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering_managers.group_id
  member_id         = aws_identitystore_user.priya_sharma.user_id
}

resource "aws_identitystore_group_membership" "derek_jones_engineering_managers" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering_managers.group_id
  member_id         = aws_identitystore_user.derek_jones.user_id
}

resource "aws_identitystore_group_membership" "lisa_kim_engineering_managers" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering_managers.group_id
  member_id         = aws_identitystore_user.lisa_kim.user_id
}

# ── Engineering (all Engineering users) ──

resource "aws_identitystore_group_membership" "james_okafor_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.james_okafor.user_id
}

resource "aws_identitystore_group_membership" "priya_sharma_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.priya_sharma.user_id
}

resource "aws_identitystore_group_membership" "derek_jones_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.derek_jones.user_id
}

resource "aws_identitystore_group_membership" "lisa_kim_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.lisa_kim.user_id
}

resource "aws_identitystore_group_membership" "alex_rivera_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.alex_rivera.user_id
}

resource "aws_identitystore_group_membership" "sam_green_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.sam_green.user_id
}

resource "aws_identitystore_group_membership" "maya_johnson_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.maya_johnson.user_id
}

resource "aws_identitystore_group_membership" "ethan_brown_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.ethan_brown.user_id
}

resource "aws_identitystore_group_membership" "chloe_wilson_engineering" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.chloe_wilson.user_id
}

# ── Platform Engineers ──

resource "aws_identitystore_group_membership" "priya_sharma_platform" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.platform_engineers.group_id
  member_id         = aws_identitystore_user.priya_sharma.user_id
}

resource "aws_identitystore_group_membership" "alex_rivera_platform" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.platform_engineers.group_id
  member_id         = aws_identitystore_user.alex_rivera.user_id
}

resource "aws_identitystore_group_membership" "sam_green_platform" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.platform_engineers.group_id
  member_id         = aws_identitystore_user.sam_green.user_id
}

# ── App Dev ──

resource "aws_identitystore_group_membership" "derek_jones_app_dev" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.app_dev.group_id
  member_id         = aws_identitystore_user.derek_jones.user_id
}

resource "aws_identitystore_group_membership" "maya_johnson_app_dev" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.app_dev.group_id
  member_id         = aws_identitystore_user.maya_johnson.user_id
}

resource "aws_identitystore_group_membership" "ethan_brown_app_dev" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.app_dev.group_id
  member_id         = aws_identitystore_user.ethan_brown.user_id
}

# ── QA Engineers ──

resource "aws_identitystore_group_membership" "lisa_kim_qa" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.qa_engineers.group_id
  member_id         = aws_identitystore_user.lisa_kim.user_id
}

resource "aws_identitystore_group_membership" "chloe_wilson_qa" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.qa_engineers.group_id
  member_id         = aws_identitystore_user.chloe_wilson.user_id
}

# ── IT & Security ──

resource "aws_identitystore_group_membership" "ryan_mitchell_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.ryan_mitchell.user_id
}

resource "aws_identitystore_group_membership" "aisha_patel_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.aisha_patel.user_id
}

resource "aws_identitystore_group_membership" "miguel_torres_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.miguel_torres.user_id
}

resource "aws_identitystore_group_membership" "tanya_brooks_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.tanya_brooks.user_id
}

resource "aws_identitystore_group_membership" "jake_hoffman_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.jake_hoffman.user_id
}

resource "aws_identitystore_group_membership" "olivia_reed_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.olivia_reed.user_id
}

resource "aws_identitystore_group_membership" "carlos_mendez_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.carlos_mendez.user_id
}

resource "aws_identitystore_group_membership" "emily_zhao_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.emily_zhao.user_id
}

resource "aws_identitystore_group_membership" "kevin_nguyen_it_security" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.it_security.group_id
  member_id         = aws_identitystore_user.kevin_nguyen.user_id
}

# ── IAM Team ──

resource "aws_identitystore_group_membership" "ryan_mitchell_iam" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.iam_team.group_id
  member_id         = aws_identitystore_user.ryan_mitchell.user_id
}

resource "aws_identitystore_group_membership" "aisha_patel_iam" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.iam_team.group_id
  member_id         = aws_identitystore_user.aisha_patel.user_id
}

resource "aws_identitystore_group_membership" "miguel_torres_iam" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.iam_team.group_id
  member_id         = aws_identitystore_user.miguel_torres.user_id
}

# ── SOC Team ──

resource "aws_identitystore_group_membership" "tanya_brooks_soc" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.soc_team.group_id
  member_id         = aws_identitystore_user.tanya_brooks.user_id
}

resource "aws_identitystore_group_membership" "jake_hoffman_soc" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.soc_team.group_id
  member_id         = aws_identitystore_user.jake_hoffman.user_id
}

resource "aws_identitystore_group_membership" "olivia_reed_soc" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.soc_team.group_id
  member_id         = aws_identitystore_user.olivia_reed.user_id
}

# ── Service Desk ──

resource "aws_identitystore_group_membership" "carlos_mendez_service_desk" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.service_desk.group_id
  member_id         = aws_identitystore_user.carlos_mendez.user_id
}

resource "aws_identitystore_group_membership" "emily_zhao_service_desk" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.service_desk.group_id
  member_id         = aws_identitystore_user.emily_zhao.user_id
}

resource "aws_identitystore_group_membership" "kevin_nguyen_service_desk" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.service_desk.group_id
  member_id         = aws_identitystore_user.kevin_nguyen.user_id
}

# ── Break Glass ──

resource "aws_identitystore_group_membership" "ryan_mitchell_break_glass" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.break_glass.group_id
  member_id         = aws_identitystore_user.ryan_mitchell.user_id
}

resource "aws_identitystore_group_membership" "aisha_patel_break_glass" {
  identity_store_id = local.identity_store_id
  group_id          = aws_identitystore_group.break_glass.group_id
  member_id         = aws_identitystore_user.aisha_patel.user_id
}
```

### AWS CLI

```powershell
$IdentityStoreId = "d-9a7b8c6d5e"

# Helper: get group ID
function Get-GroupId($displayName) {
  aws identitystore list-groups --identity-store-id $IdentityStoreId `
    --filter AttributePath=DisplayName,AttributeValue=$displayName `
    --query "Groups[0].GroupId" --output text
}

# Helper: get user ID
function Get-UserId($userName) {
  aws identitystore list-users --identity-store-id $IdentityStoreId `
    --filter AttributePath=UserName,AttributeValue=$userName `
    --query "Users[0].UserId" --output text
}

# Add user to group
function Add-ToGroup($userEmail, $groupName) {
  $groupId = Get-GroupId $groupName
  $userId = Get-UserId $userEmail
  aws identitystore create-group-membership `
    --identity-store-id $IdentityStoreId `
    --group-id $groupId `
    --member-id $userId
  Write-Host "Added $userEmail to $groupName"
}

# Engineering managers
Add-ToGroup "james.okafor@innogrid.com" "engineering-managers"
Add-ToGroup "priya.sharma@innogrid.com" "engineering-managers"
Add-ToGroup "derek.jones@innogrid.com" "engineering-managers"
Add-ToGroup "lisa.kim@innogrid.com" "engineering-managers"

# Engineering (all engineers)
Add-ToGroup "james.okafor@innogrid.com" "engineering"
Add-ToGroup "priya.sharma@innogrid.com" "engineering"
Add-ToGroup "derek.jones@innogrid.com" "engineering"
Add-ToGroup "lisa.kim@innogrid.com" "engineering"
Add-ToGroup "alex.rivera@innogrid.com" "engineering"
Add-ToGroup "sam.green@innogrid.com" "engineering"
Add-ToGroup "maya.johnson@innogrid.com" "engineering"
Add-ToGroup "ethan.brown@innogrid.com" "engineering"
Add-ToGroup "chloe.wilson@innogrid.com" "engineering"

# Platform engineers
Add-ToGroup "priya.sharma@innogrid.com" "platform-engineers"
Add-ToGroup "alex.rivera@innogrid.com" "platform-engineers"
Add-ToGroup "sam.green@innogrid.com" "platform-engineers"

# App dev
Add-ToGroup "derek.jones@innogrid.com" "app-dev"
Add-ToGroup "maya.johnson@innogrid.com" "app-dev"
Add-ToGroup "ethan.brown@innogrid.com" "app-dev"

# QA engineers
Add-ToGroup "lisa.kim@innogrid.com" "qa-engineers"
Add-ToGroup "chloe.wilson@innogrid.com" "qa-engineers"

# IT & Security
Add-ToGroup "ryan.mitchell@innogrid.com" "it-security"
Add-ToGroup "aisha.patel@innogrid.com" "it-security"
Add-ToGroup "miguel.torres@innogrid.com" "it-security"
Add-ToGroup "tanya.brooks@innogrid.com" "it-security"
Add-ToGroup "jake.hoffman@innogrid.com" "it-security"
Add-ToGroup "olivia.reed@innogrid.com" "it-security"
Add-ToGroup "carlos.mendez@innogrid.com" "it-security"
Add-ToGroup "emily.zhao@innogrid.com" "it-security"
Add-ToGroup "kevin.nguyen@innogrid.com" "it-security"

# IAM team
Add-ToGroup "ryan.mitchell@innogrid.com" "iam-team"
Add-ToGroup "aisha.patel@innogrid.com" "iam-team"
Add-ToGroup "miguel.torres@innogrid.com" "iam-team"

# SOC team
Add-ToGroup "tanya.brooks@innogrid.com" "soc-team"
Add-ToGroup "jake.hoffman@innogrid.com" "soc-team"
Add-ToGroup "olivia.reed@innogrid.com" "soc-team"

# Service desk
Add-ToGroup "carlos.mendez@innogrid.com" "service-desk"
Add-ToGroup "emily.zhao@innogrid.com" "service-desk"
Add-ToGroup "kevin.nguyen@innogrid.com" "service-desk"

# Break glass
Add-ToGroup "ryan.mitchell@innogrid.com" "break-glass"
Add-ToGroup "aisha.patel@innogrid.com" "break-glass"
```

### AWS Console

1. **AWS Console** → IAM Identity Centre → **Groups** → Select group (e.g., `engineering`)
2. Click **Add user** → Select all engineering user checkboxes → **Add user**
3. Repeat for each group following the user-to-group mapping in `design.md`

---

## 6. Account Assignments

### Terraform

```hcl
# terraform/identity-centre/account-assignments.tf

locals {
  instance_arn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
  nonprod_id   = "123456789012"
  security_id  = "444455556666"
  mgmt_id      = "111122223333"
}

# ── Engineering → DevAccess → Nonproduction ──

resource "aws_ssoadmin_account_assignment" "engineering_devaccess_nonprod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.engineering.group_id
  target_id          = local.nonprod_id
  target_type        = "AWS_ACCOUNT"
}

resource "aws_ssoadmin_account_assignment" "platform_engineers_devaccess_nonprod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.platform_engineers.group_id
  target_id          = local.nonprod_id
  target_type        = "AWS_ACCOUNT"
}

resource "aws_ssoadmin_account_assignment" "app_dev_devaccess_nonprod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.app_dev.group_id
  target_id          = local.nonprod_id
  target_type        = "AWS_ACCOUNT"
}

resource "aws_ssoadmin_account_assignment" "qa_engineers_devaccess_nonprod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.qa_engineers.group_id
  target_id          = local.nonprod_id
  target_type        = "AWS_ACCOUNT"
}

# ── IT Security → ReadOnly → Security ──

resource "aws_ssoadmin_account_assignment" "it_security_readonly_security" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.read_only.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.it_security.group_id
  target_id          = local.security_id
  target_type        = "AWS_ACCOUNT"
}

# ── IAM Team → AdministratorAccess → Management ──

resource "aws_ssoadmin_account_assignment" "iam_team_admin_mgmt" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.admin_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.iam_team.group_id
  target_id          = local.mgmt_id
  target_type        = "AWS_ACCOUNT"
}

# ── SOC Team → SecurityAudit → Security ──

resource "aws_ssoadmin_account_assignment" "soc_team_securityaudit_security" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.security_audit.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.soc_team.group_id
  target_id          = local.security_id
  target_type        = "AWS_ACCOUNT"
}

# ── Service Desk → ReadOnly → Nonproduction ──

resource "aws_ssoadmin_account_assignment" "service_desk_readonly_nonprod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.read_only.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.service_desk.group_id
  target_id          = local.nonprod_id
  target_type        = "AWS_ACCOUNT"
}

# ── Break Glass → AdministratorAccess → All accounts ──

resource "aws_ssoadmin_account_assignment" "break_glass_admin_mgmt" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.admin_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.break_glass.group_id
  target_id          = local.mgmt_id
  target_type        = "AWS_ACCOUNT"
}

resource "aws_ssoadmin_account_assignment" "break_glass_admin_security" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.admin_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.break_glass.group_id
  target_id          = local.security_id
  target_type        = "AWS_ACCOUNT"
}

resource "aws_ssoadmin_account_assignment" "break_glass_admin_nonprod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.admin_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.break_glass.group_id
  target_id          = local.nonprod_id
  target_type        = "AWS_ACCOUNT"
}

resource "aws_ssoadmin_account_assignment" "break_glass_admin_prod" {
  instance_arn       = local.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.admin_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.break_glass.group_id
  target_id          = "777788889999"
  target_type        = "AWS_ACCOUNT"
}
```

### AWS CLI

```powershell
$InstanceArn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
$NonprodId = "123456789012"
$SecurityId = "444455556666"
$MgmtId = "111122223333"
$ProdId = "777788889999"
$IdentityStoreId = "d-9a7b8c6d5e"

function Assign-GroupToAccount($groupName, $permissionSetName, $accountId) {
  $groupId = aws identitystore list-groups --identity-store-id $IdentityStoreId `
    --filter AttributePath=DisplayName,AttributeValue=$groupName `
    --query "Groups[0].GroupId" --output text

  $psArn = aws sso-admin list-permission-sets --instance-arn $InstanceArn `
    --query "PermissionSets[?contains(@, '$permissionSetName')]" --output text

  aws sso-admin create-account-assignment `
    --instance-arn $InstanceArn `
    --permission-set-arn $psArn `
    --principal-type GROUP `
    --principal-id $groupId `
    --target-id $accountId `
    --target-type AWS_ACCOUNT

  Write-Host "Assigned $groupName → $permissionSetName → Account $accountId"
}

# Assignments
Assign-GroupToAccount "engineering" "DevAccess" $NonprodId
Assign-GroupToAccount "platform-engineers" "DevAccess" $NonprodId
Assign-GroupToAccount "app-dev" "DevAccess" $NonprodId
Assign-GroupToAccount "qa-engineers" "DevAccess" $NonprodId
Assign-GroupToAccount "it-security" "ReadOnly" $SecurityId
Assign-GroupToAccount "iam-team" "AdministratorAccess" $MgmtId
Assign-GroupToAccount "soc-team" "SecurityAudit" $SecurityId
Assign-GroupToAccount "service-desk" "ReadOnly" $NonprodId
Assign-GroupToAccount "break-glass" "AdministratorAccess" $MgmtId
Assign-GroupToAccount "break-glass" "AdministratorAccess" $SecurityId
Assign-GroupToAccount "break-glass" "AdministratorAccess" $NonprodId
Assign-GroupToAccount "break-glass" "AdministratorAccess" $ProdId
```

### AWS Console

1. **AWS Console** → IAM Identity Centre → **AWS accounts**
2. Click an account (e.g., `inno-nonprod`)
3. Click **Assign users or groups**
4. Select the **Groups** tab
5. Check the group (e.g., `engineering`)
6. Select the permission set (e.g., `DevAccess`)
7. Click **Submit**
8. Repeat for each assignment in the matrix:

| Group | Permission Set | Account |
|---|---|---|
| `engineering` | `DevAccess` | Nonproduction |
| `platform-engineers` | `DevAccess` | Nonproduction |
| `app-dev` | `DevAccess` | Nonproduction |
| `qa-engineers` | `DevAccess` | Nonproduction |
| `it-security` | `ReadOnly` | Security |
| `iam-team` | `AdministratorAccess` | Management |
| `soc-team` | `SecurityAudit` | Security |
| `service-desk` | `ReadOnly` | Nonproduction |
| `break-glass` | `AdministratorAccess` | All 4 accounts |

---

## 7. SCIM Configuration (Entra ID → IAM Identity Centre)

### Step 1: Enable SCIM in IAM Identity Centre

1. **AWS Console** → IAM Identity Centre → **Settings** → **Identity source** → **Actions** → **Manage provisioning**
2. Note the **SCIM endpoint** URL and **Access token** (click **Generate new token**)
3. Save these values — they are needed for the Entra ID configuration

### Step 2: Configure Entra ID

1. Sign in to **Entra ID admin center** as a Global Administrator
2. Navigate to **Enterprise applications** → **New application** → **Create your own application**
3. Name: `AWS IAM Identity Centre (InnoGrid)`
4. Select **Integrate any other application you don't find in the gallery**
5. Under **Manage** → **Provisioning** → **Get started**
6. Set **Provisioning mode** to **Automatic**
7. Enter:
   - **Tenant URL**: (SCIM endpoint from Step 1)
   - **Secret token**: (Access token from Step 1)
8. Click **Test Connection**
9. Save the configuration

### Step 3: Attribute Mapping

1. Under **Provisioning** → **Mappings**, ensure the default mappings are correct:
   - `userPrincipalName` → `userName`
   - `displayName` → `displayName`
   - `givenName` → `firstName`
   - `sn` → `lastName`
   - `mail` → `emails[type eq "work"].value`
2. Click **Save**

### Step 4: Set Scope and Start Provisioning

1. Under **Provisioning** → **Settings**:
   - **Scope**: `Sync only assigned users and groups`
2. Under **Users and groups**, add the Entra ID groups that should sync (HR, Finance, Legal, Exec)
3. Click **Start provisioning**
4. Initial sync typically completes within 10–40 minutes

---

## 8. Terraform Backend Setup

### S3 Bucket + DynamoDB for State Management

```hcl
# terraform/backend-setup/main.tf

terraform {
  required_version = ">= 1.5"
}

provider "aws" {
  region = "eu-west-2"
}

# S3 bucket for Terraform state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "inno-terraform-state-111122223333"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB for state locking
resource "aws_dynamodb_table" "terraform_state_lock" {
  name         = "inno-terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Terraform Backend Configuration

```hcl
# terraform/identity-centre/backend.tf

terraform {
  backend "s3" {
    bucket         = "inno-terraform-state-111122223333"
    key            = "identity-centre/terraform.tfstate"
    region         = "eu-west-2"
    dynamodb_table = "inno-terraform-state-lock"
    encrypt        = true
  }
}
```

### Deployment

```powershell
# Step 1: Deploy backend infrastructure
Set-Location "C:\terraform\backend-setup"
terraform init
terraform apply -auto-approve

# Step 2: Deploy Identity Centre resources
Set-Location "C:\terraform\identity-centre"
terraform init -reconfigure
terraform plan -out=ic-plan.tfplan
terraform apply ic-plan.tfplan
```

---

## 9. Verification

### Verify Users

```powershell
$IdentityStoreId = "d-9a7b8c6d5e"

# List all users
aws identitystore list-users --identity-store-id $IdentityStoreId `
  --query "Users[*].[UserName,DisplayName,UserType]" --output table

# Expected: 18 users
aws identitystore list-users --identity-store-id $IdentityStoreId `
  --query "length(Users)"
# Expected output: 18
```

### Verify Groups

```powershell
# List all groups
aws identitystore list-groups --identity-store-id $IdentityStoreId `
  --query "Groups[*].[DisplayName,Description]" --output table

# Expected: 10 groups
aws identitystore list-groups --identity-store-id $IdentityStoreId `
  --query "length(Groups)"
# Expected output: 10
```

### Verify Group Memberships

```powershell
# Check members of engineering group
$EngineeringGroupId = aws identitystore list-groups --identity-store-id $IdentityStoreId `
  --filter AttributePath=DisplayName,AttributeValue=engineering `
  --query "Groups[0].GroupId" --output text

aws identitystore list-group-memberships --identity-store-id $IdentityStoreId `
  --group-id $EngineeringGroupId --query "length(GroupMemberships)"
# Expected: 9 members

# Check members of iam-team
$IamTeamGroupId = aws identitystore list-groups --identity-store-id $IdentityStoreId `
  --filter AttributePath=DisplayName,AttributeValue=iam-team `
  --query "Groups[0].GroupId" --output text

aws identitystore list-group-memberships --identity-store-id $IdentityStoreId `
  --group-id $IamTeamGroupId --query "GroupMemberships[*].MemberId"
# Expected: Ryan Mitchell, Aisha Patel, Miguel Torres
```

### Verify Permission Sets

```powershell
$InstanceArn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"

# List all permission sets
aws sso-admin list-permission-sets --instance-arn $InstanceArn `
  --query "PermissionSets[*]" --output table

# Describe DevAccess
$DevAccessArn = aws sso-admin list-permission-sets --instance-arn $InstanceArn `
  --query "PermissionSets[?contains(@, 'DevAccess')]" --output text

aws sso-admin describe-permission-set `
  --instance-arn $InstanceArn `
  --permission-set-arn $DevAccessArn `
  --query "PermissionSet.[Name,SessionDuration,RelayState]"
# Expected: "DevAccess", "PT8H", "https://console.aws.amazon.com/ec2"
```

### Verify Account Assignments

```powershell
# List assignments for Nonproduction account
aws sso-admin list-account-assignments `
  --instance-arn $InstanceArn `
  --account-id 123456789012 `
  --query "AccountAssignments[*].[PrincipalType,PrincipalId,PermissionSetArn]" --output table

# Expected: engineering, platform-engineers, app-dev, qa-engineers, service-desk, break-glass
```

### End-to-End Test

```powershell
# Generate a sign-in URL for the AWS access portal
# (This is specific to the IAM Identity Centre instance)
$PortalUrl = "https://d-9a7b8c6d5e.awsapps.com/start"
Write-Host "Access portal URL: $PortalUrl"

# Instructions:
# 1. Open the portal URL in a browser
# 2. Sign in as a test user (e.g., alex.rivera@innogrid.com)
# 3. Verify the user sees the Nonproduction account with DevAccess permission set
# 4. Sign in as ryan.mitchell@innogrid.com
# 5. Verify the user sees Management account with AdministratorAccess
```

---

## Summary

| Resource | Count | Details |
|---|---|---|
| Groups | 10 | `engineering`, `platform-engineers`, `app-dev`, `qa-engineers`, `engineering-managers`, `it-security`, `iam-team`, `soc-team`, `service-desk`, `break-glass` |
| Permission Sets | 5 | `DevAccess`, `ReadOnly`, `SecurityAudit`, `AdministratorAccess`, `PowerUserAccess` |
| Users | 18 | All active Engineering & IT/Security personnel |
| Group Memberships | ~40 | Mapped per user-to-group matrix |
| Account Assignments | 12 | Group → permission set → account mappings |

The foundation is now complete. The IAM Identity Centre is fully initialised with groups, permission sets, users, and account assignments. Subsequent scenarios (1–6) can now operate on this established environment.
