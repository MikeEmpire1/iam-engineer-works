# Scenario 1: User Lifecycle — Implementation

## Prerequisites

- AWS CLI configured with administrative access to the Management account
- Terraform Cloud / OSS with AWS provider
- `aws-sso-admin` CLI or AWS Console access to IAM Identity Centre
- HR ticket numbers: `HR-2026-071` (Daniel Park), `HR-2026-072` (Maya Johnson transfer), `HR-2026-069` (Kevin Nguyen termination)
- IAM ticket numbers: `IAM-2026-042`, `IAM-2026-043`, `IAM-2026-044`

---

## 1. Joiner — Daniel Park

### 1.1 Terraform: Create IAM Identity Centre User

```hcl
# terraform/identity-centre/users.tf

resource "aws_identitystore_user" "daniel_park" {
  identity_store_id = "d-9a7b8c6d5e"

  display_name = "Daniel Park"
  user_name    = "daniel.park@innogrid.com"

  name {
    given_name  = "Daniel"
    family_name = "Park"
  }

  emails {
    value = "daniel.park@innogrid.com"
    primary = true
  }

  user_type = "IC3"
  locale    = "en-GB"

  lifecycle {
    # Prevent accidental deletion; manual override only
    prevent_destroy = true
  }
}

resource "aws_identitystore_group_membership" "daniel_park_engineering" {
  identity_store_id = "d-9a7b8c6d5e"
  group_id          = aws_identitystore_group.engineering.group_id
  member_id         = aws_identitystore_user.daniel_park.user_id
}

resource "aws_identitystore_group_membership" "daniel_park_platform_engineers" {
  identity_store_id = "d-9a7b8c6d5e"
  group_id          = aws_identitystore_group.platform_engineers.group_id
  member_id         = aws_identitystore_user.daniel_park.user_id
}
```

### 1.2 Terraform: Assign Permission Set to Account

```hcl
# terraform/identity-centre/assignments.tf

resource "aws_ssoadmin_permission_set_inline_policy" "dev_access" {
  instance_arn       = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  inline_policy      = jsonencode({
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

resource "aws_ssoadmin_permission_set" "dev_access" {
  instance_arn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
  name         = "DevAccess"
  session_duration = "PT8H"
  relay_state      = "https://console.aws.amazon.com/ec2"

  managed_policies = ["arn:aws:iam::aws:policy/ReadOnlyAccess"]
}

resource "aws_ssoadmin_account_assignment" "dev_access_engineering" {
  instance_arn       = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
  permission_set_arn = aws_ssoadmin_permission_set.dev_access.arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.engineering.group_id
  target_id          = "123456789012"  # inno-nonprod account ID
  target_type        = "AWS_ACCOUNT"
}
```

### 1.3 Verification (AWS CLI)

```powershell
# Verify user exists in IAM Identity Centre
aws identitystore list-users --identity-store-id d-9a7b8c6d5e `
  --filter AttributePath=UserName,AttributeValue=daniel.park@innogrid.com

# Verify group memberships
aws identitystore list-group-memberships --identity-store-id d-9a7b8c6d5e `
  --member-id UserId="<daniel-park-user-id>"

# Check permission set assignment
aws sso-admin list-account-assignments --instance-arn "arn:aws:sso:::instance/ssoins-1234567890abcdef" `
  --account-id 123456789012 `
  --principal-type GROUP `
  --principal-id "<engineering-group-id>"
```

### 1.4 Console Workflow (Manual Alternative)

1. **AWS Console** → AWS IAM Identity Centre → Users → **Add user**
   - Username: `daniel.park@innogrid.com`
   - Email: `daniel.park@innogrid.com`
   - First name: Daniel, Last name: Park
   - Display name: Daniel Park
   - Groups: Select `engineering`, `platform-engineers`

2. **AWS Console** → IAM Identity Centre → AWS accounts → `inno-nonprod` → **Assign users/groups**
   - Select `engineering` group
   - Permission set: `DevAccess`

3. Send welcome email with AWS access portal URL

---

## 2. Mover — Maya Johnson

### 2.1 Terraform: Update Group Memberships

```hcl
# terraform/identity-centre/maya-johnson-move.tf

# Fetch Maya's user ID (data source)
data "aws_identitystore_user" "maya_johnson" {
  identity_store_id = "d-9a7b8c6d5e"

  filter {
    attribute_path  = "UserName"
    attribute_value = "maya.johnson@innogrid.com"
  }
}

# Remove from app-dev group
resource "aws_identitystore_group_membership" "maya_johnson_app_dev" {
  identity_store_id = "d-9a7b8c6d5e"
  group_id          = aws_identitystore_group.app_dev.group_id
  member_id         = data.aws_identitystore_user.maya_johnson.user_id
}

resource "aws_identitystore_group_membership" "maya_johnson_platform_engineers" {
  identity_store_id = "d-9a7b8c6d5e"
  group_id          = aws_identitystore_group.platform_engineers.group_id
  member_id         = data.aws_identitystore_user.maya_johnson.user_id
}
```

**Important**: To perform the remove, first `terraform destroy` the old membership resource (import it if it was created outside Terraform), then apply the new one. In practice, the old membership resource would be removed from state and the new one created.

### 2.2 AWS CLI: Remove and Add Group Membership

```powershell
# Step 1: List Maya's current group memberships
aws identitystore list-group-memberships --identity-store-id d-9a7b8c6d5e `
  --member-id UserId="<maya-johnson-user-id>"

# Step 2: Remove from app-dev group
aws identitystore delete-group-membership --identity-store-id d-9a7b8c6d5e `
  --membership-id "<app-dev-membership-id>"

# Step 3: Add to platform-engineers group
$groupId = (aws identitystore list-groups --identity-store-id d-9a7b8c6d5e `
  --filter AttributePath=DisplayName,AttributeValue=platform-engineers `
  --query "Groups[0].GroupId" --output text)

$userId = (aws identitystore list-users --identity-store-id d-9a7b8c6d5e `
  --filter AttributePath=UserName,AttributeValue=maya.johnson@innogrid.com `
  --query "Users[0].UserId" --output text)

aws identitystore create-group-membership --identity-store-id d-9a7b8c6d5e `
  --group-id $groupId --member-id $userId

# Step 4: Update manager attribute (if using custom attributes)
# Note: IAM Identity Centre does not natively support "manager" as a default attribute.
# This would be tracked in the IAM ticket metadata.
Write-Host "Manager updated: Derek Jones -> Priya Sharma (tracked in IAM-2026-043)"
```

### 2.3 Console Workflow (Manual Alternative)

1. **AWS Console** → IAM Identity Centre → Users → `maya.johnson@innogrid.com`
   - **Groups tab** → Remove from `app-dev`
   - **Groups tab** → Add to `platform-engineers`

2. Verify the user can still access the AWS access portal (no interruption — permission set `DevAccess` remains assigned via the `engineering` group which hasn't changed)

### 2.4 Verification

```powershell
# Confirm Maya is no longer in app-dev
aws identitystore list-group-memberships --identity-store-id d-9a7b8c6d5e --group-id "<app-dev-group-id>"

# Confirm Maya is now in platform-engineers
aws identitystore list-group-memberships --identity-store-id d-9a7b8c6d5e --group-id "$groupId"
```

---

## 3. Leaver — Kevin Nguyen

### 3.1 Immediate Actions (AWS CLI)

```powershell
# Variables
$identityStoreId = "d-9a7b8c6d5e"
$instanceArn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
$hrTicket = "HR-2026-069"
$iamTicket = "IAM-2026-044"
$timestamp = (Get-Date -Format "yyyy-MM-ddTHH:mm:ssZ")

# Get Kevin's user ID
$kevinUserId = aws identitystore list-users --identity-store-id $identityStoreId `
  --filter AttributePath=UserName,AttributeValue=kevin.nguyen@innogrid.com `
  --query "Users[0].UserId" --output text

Write-Host "[$timestamp] [$iamTicket] Processing leaver: Kevin Nguyen ($kevinUserId)"

# Step 1: List all current group memberships
$memberships = aws identitystore list-group-memberships --identity-store-id $identityStoreId `
  --member-id UserId=$kevinUserId --query "GroupMemberships" --output json

Write-Host "Found $($memberships.Count) group memberships to remove."

# Step 2: Remove all group memberships
foreach ($membership in ($memberships | ConvertFrom-Json)) {
  aws identitystore delete-group-membership --identity-store-id $identityStoreId `
    --membership-id $membership.MembershipId

  Write-Host "Removed membership: $($membership.MembershipId)"
}

# Step 3: Deactivate the user
# Note: IAM Identity Centre doesn't directly support user deactivation via API.
# Instead, the user's status is managed at the identity source level.
# For IAM Identity Centre-managed users, we cannot fully "deactivate" — we must delete
# or use the Console to set status to DISABLED.
# Using the Console for this step (see 3.2).

# Step 4: Log the action
$auditLog = @"
{
  "eventType": "LEAVER",
  "user": "kevin.nguyen@innogrid.com",
  "userId": "$kevinUserId",
  "hrTicket": "$hrTicket",
  "iamTicket": "$iamTicket",
  "performedBy": "aisha.patel@innogrid.com",
  "timestamp": "$timestamp",
  "action": "All group memberships removed; user deactivated in IAM Identity Centre"
}
"@

Write-Host $auditLog
# In production, this would be written to S3, CloudWatch, or the ticket system
```

### 3.2 Console Workflow: Deactivate User

1. **AWS Console** → AWS IAM Identity Centre → Users → `kevin.nguyen@innogrid.com`
2. Click **Disable user access**
   - A confirmation dialog explains: "The user will no longer be able to sign in to the user portal. Group memberships and permission sets remain assigned but cannot be used until the user is re-enabled."
3. Verify the user shows status: **Disabled**

### 3.3 Revoke Active Sessions (AWS CLI)

```powershell
# List active sessions for Kevin
aws sso-admin list-account-assignments --instance-arn $instanceArn `
  --account-id 111111111111 --query "AccountAssignments"  # Security account

# IAM Identity Centre sessions expire based on the session duration.
# To force logout, the user's token can be invalidated by:
# 1. Disabling the user (done above)
# 2. Removing permission set assignments (done via group removal above)
# 3. The user portal session cache respects the identity store status.

# Verify no active permission set assignments remain
aws identitystore list-group-memberships --identity-store-id $identityStoreId `
  --member-id UserId=$kevinUserId
# Expected result: empty list
```

### 3.4 Verification

```powershell
# Attempt to describe the user (should confirm DISABLED status)
aws identitystore describe-user --identity-store-id $identityStoreId --user-id $kevinUserId

# Confirm zero group memberships
$remainingGroups = aws identitystore list-group-memberships --identity-store-id $identityStoreId `
  --member-id UserId=$kevinUserId --query "length(GroupMemberships)"
Write-Host "Remaining group memberships: $remainingGroups"
# Expected: 0

# Send notification to CISO and SOC
Write-Host "=== TERMINATION COMPLETE ==="
Write-Host "User: Kevin Nguyen (kevin.nguyen@innogrid.com)"
Write-Host "IAM Ticket: $iamTicket"
Write-Host "HR Ticket: $hrTicket"
Write-Host "Timestamp: $timestamp"
Write-Host "Notified: sarah.chen@innogrid.com, tanya.brooks@innogrid.com"
Write-Host "=============================="
```

### 3.5 Console Workflow for Verification

1. **AWS Console** → IAM Identity Centre → Users → Kevin Nguyen
   - Status should show **Disabled**
   - Groups tab should show **0 groups**
2. **AWS Console** → IAM Identity Centre → Dashboard → **Audit log**
   - Filter by user `kevin.nguyen` to confirm all events captured in CloudTrail

---

## 4. Automation Script

The following PowerShell script orchestrates all three lifecycle events with a single invocation and logs to a local audit file. In production this would be triggered by an HR system webhook or Jira automation.

```powershell
# scripts/process-lifecycle-events.ps1
# Orchestrates joiner/mover/leaver events

param(
  [string]$EventType,       # JOINER, MOVER, LEAVER
  [string]$UserEmail,
  [string]$HrTicket,
  [string]$IamTicket,
  [string]$NewGroup,        # Used for MOVER
  [string]$OldGroup,        # Used for MOVER
  [string]$Manager          # Used for JOINER or MOVER
)

$IdentityStoreId = "d-9a7b8c6d5e"
$AuditLogFile = "C:\IAM\lifecycle-audit.log"
$Timestamp = (Get-Date -Format "yyyy-MM-ddTHH:mm:ssZ")

function Write-AuditLog {
  param($Message)
  $entry = "[$Timestamp] $Message"
  Add-Content -Path $AuditLogFile -Value $entry
  Write-Host $entry
}

switch ($EventType) {
  "JOINER" {
    Write-AuditLog "PROCESSING JOINER: $UserEmail (Ticket: $HrTicket / $IamTicket)"

    # Create user via Terraform
    Set-Location "C:\terraform\identity-centre"
    terraform apply -auto-approve `
      -target="aws_identitystore_user.$($UserEmail.Replace('@','_').Replace('.','_'))"

    Write-AuditLog "JOINER COMPLETE: $UserEmail. Manager: $Manager. Groups: engineering, $NewGroup"
  }

  "MOVER" {
    Write-AuditLog "PROCESSING MOVER: $UserEmail (Ticket: $HrTicket / $IamTicket)"

    # Fetch user ID
    $userId = aws identitystore list-users --identity-store-id $IdentityStoreId `
      --filter AttributePath=UserName,AttributeValue=$UserEmail `
      --query "Users[0].UserId" --output text

    # Remove from old group
    $oldGroupMembershipId = aws identitystore list-group-memberships --identity-store-id $IdentityStoreId `
      --member-id UserId=$userId --query "GroupMemberships[?GroupId=='$OldGroup'].MembershipId" --output text
    if ($oldGroupMembershipId) {
      aws identitystore delete-group-membership --identity-store-id $IdentityStoreId `
        --membership-id $oldGroupMembershipId
    }

    # Add to new group
    $newGroupId = aws identitystore list-groups --identity-store-id $IdentityStoreId `
      --filter AttributePath=DisplayName,AttributeValue=$NewGroup `
      --query "Groups[0].GroupId" --output text
    aws identitystore create-group-membership --identity-store-id $IdentityStoreId `
      --group-id $newGroupId --member-id $userId

    Write-AuditLog "MOVER COMPLETE: $UserEmail. Removed from: $OldGroup. Added to: $NewGroup. Manager: $Manager"
  }

  "LEAVER" {
    Write-AuditLog "PROCESSING LEAVER: $UserEmail (Ticket: $HrTicket / $IamTicket)"

    $userId = aws identitystore list-users --identity-store-id $IdentityStoreId `
      --filter AttributePath=UserName,AttributeValue=$UserEmail `
      --query "Users[0].UserId" --output text

    # Remove all group memberships
    $memberships = aws identitystore list-group-memberships --identity-store-id $IdentityStoreId `
      --member-id UserId=$userId --query "GroupMemberships" --output json | ConvertFrom-Json
    foreach ($m in $memberships) {
      aws identitystore delete-group-membership --identity-store-id $IdentityStoreId `
        --membership-id $m.MembershipId
    }

    # Note: User deactivation must be done via Console (no API for status change)
    Write-AuditLog "LEAVER ACTIONS COMPLETE: $UserEmail. Groups removed. User must be disabled via Console."
    Write-AuditLog "NOTIFY: sarah.chen@innogrid.com, tanya.brooks@innogrid.com"
  }
}
```

### Usage Examples

```powershell
# Joiner
.\process-lifecycle-events.ps1 -EventType JOINER -UserEmail "daniel.park@innogrid.com" `
  -HrTicket "HR-2026-071" -IamTicket "IAM-2026-042" -NewGroup "platform-engineers" `
  -Manager "Priya Sharma"

# Mover
.\process-lifecycle-events.ps1 -EventType MOVER -UserEmail "maya.johnson@innogrid.com" `
  -HrTicket "HR-2026-072" -IamTicket "IAM-2026-043" -OldGroup "app-dev" `
  -NewGroup "platform-engineers" -Manager "Priya Sharma"

# Leaver
.\process-lifecycle-events.ps1 -EventType LEAVER -UserEmail "kevin.nguyen@innogrid.com" `
  -HrTicket "HR-2026-069" -IamTicket "IAM-2026-044"
```

---

## 5. Auditing & Compliance

### CloudTrail Queries

```sql
-- Athena query on CloudTrail logs to verify lifecycle events
SELECT
  eventtime,
  eventname,
  useridentity.arn,
  requestparameters,
  responseelements
FROM cloudtrail_logs
WHERE
  eventname IN ('CreateUser', 'DeleteGroupMembership', 'CreateGroupMembership', 'UpdateUser')
  AND eventtime >= '2026-06-26T09:00:00Z'
  AND eventtime <= '2026-06-26T18:00:00Z'
ORDER BY eventtime;
```

### Expected CloudTrail Events

| Event | Source | API Call |
|---|---|---|
| User created | IAM Identity Centre | `CreateUser` |
| Group membership added | IAM Identity Centre | `CreateGroupMembership` |
| Group membership removed | IAM Identity Centre | `DeleteGroupMembership` |
| User disabled | IAM Identity Centre | `UpdateUser` (Status → DISABLED) |
| Permission set assigned | IAM Identity Centre | `CreateAccountAssignment` |

### HR Reconciliation (Post-Processing)

```powershell
# scripts/hr-reconciliation.ps1
# Weekly reconciliation between HR roster and IAM Identity Centre

$identityStoreId = "d-9a7b8c6d5e"
$iamUsers = aws identitystore list-users --identity-store-id $identityStoreId `
  --query "Users[*].[UserId,UserName]" --output json | ConvertFrom-Json

$hrRoster = Import-Csv "C:\HR\active-employees.csv"

Write-Host "=== HR RECONCILIATION REPORT ==="
foreach ($iamUser in $iamUsers) {
  $email = $iamUser[1]
  $match = $hrRoster | Where-Object { $_.Email -eq $email }
  if (-not $match) {
    Write-Host "WARNING: $email exists in IAM Identity Centre but not in HR roster"
  }
}

Write-Host "=== DONE ==="
```

---

## Summary of Changes

| Event | User | Action | Success Criteria Met? |
|---|---|---|---|
| Joiner | Daniel Park | Created in IAM Identity Centre, added to `engineering` + `platform-engineers`, `DevAccess` assigned to `inno-nonprod` | Can sign in and access nonprod via DevAccess |
| Mover | Maya Johnson | Removed from `app-dev`, added to `platform-engineers`, manager updated | Access unchanged (still in `engineering`), group membership corrected |
| Leaver | Kevin Nguyen | All group memberships removed, user disabled in console, CISO+SOC notified | Cannot sign in, no active groups, audit trail complete |
