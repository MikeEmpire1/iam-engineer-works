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

```bash
IDENTITY_STORE_ID="d-9a7b8c6d5e"
INSTANCE_ARN="arn:aws:sso:::instance/ssoins-1234567890abcdef"

# Verify user exists in IAM Identity Centre
aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID" \
  --filter "AttributePath=UserName,AttributeValue=daniel.park@innogrid.com"

# Verify group memberships
aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
  --member-id "UserId=<daniel-park-user-id>"

# Check permission set assignment
aws sso-admin list-account-assignments --instance-arn "$INSTANCE_ARN" \
  --account-id 123456789012 \
  --principal-type GROUP \
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

```bash
IDENTITY_STORE_ID="d-9a7b8c6d5e"

# Step 1: List Maya's current group memberships
aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
  --member-id "UserId=<maya-johnson-user-id>"

# Step 2: Remove from app-dev group
aws identitystore delete-group-membership --identity-store-id "$IDENTITY_STORE_ID" \
  --membership-id "<app-dev-membership-id>"

# Step 3: Add to platform-engineers group
GROUP_ID=$(aws identitystore list-groups --identity-store-id "$IDENTITY_STORE_ID" \
  --filter "AttributePath=DisplayName,AttributeValue=platform-engineers" \
  --query "Groups[0].GroupId" --output text)

USER_ID=$(aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID" \
  --filter "AttributePath=UserName,AttributeValue=maya.johnson@innogrid.com" \
  --query "Users[0].UserId" --output text)

aws identitystore create-group-membership --identity-store-id "$IDENTITY_STORE_ID" \
  --group-id "$GROUP_ID" --member-id "$USER_ID"

# Step 4: Update manager attribute (if using custom attributes)
# Note: IAM Identity Centre does not natively support "manager" as a default attribute.
# This would be tracked in the IAM ticket metadata.
echo "Manager updated: Derek Jones -> Priya Sharma (tracked in IAM-2026-043)"
```

### 2.3 Console Workflow (Manual Alternative)

1. **AWS Console** → IAM Identity Centre → Users → `maya.johnson@innogrid.com`
   - **Groups tab** → Remove from `app-dev`
   - **Groups tab** → Add to `platform-engineers`

2. Verify the user can still access the AWS access portal (no interruption — permission set `DevAccess` remains assigned via the `engineering` group which hasn't changed)

### 2.4 Verification

```bash
IDENTITY_STORE_ID="d-9a7b8c6d5e"

# Confirm Maya is no longer in app-dev
aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" --group-id "<app-dev-group-id>"

# Confirm Maya is now in platform-engineers
aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" --group-id "$GROUP_ID"
```

---

## 3. Leaver — Kevin Nguyen

### 3.1 Immediate Actions (AWS CLI)

```bash
IDENTITY_STORE_ID="d-9a7b8c6d5e"
INSTANCE_ARN="arn:aws:sso:::instance/ssoins-1234567890abcdef"
HR_TICKET="HR-2026-069"
IAM_TICKET="IAM-2026-044"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Get Kevin's user ID
KEVIN_USER_ID=$(aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID" \
  --filter "AttributePath=UserName,AttributeValue=kevin.nguyen@innogrid.com" \
  --query "Users[0].UserId" --output text)

echo "[$TIMESTAMP] [$IAM_TICKET] Processing leaver: Kevin Nguyen ($KEVIN_USER_ID)"

# Step 1: List all current group memberships
MEMBERSHIPS=$(aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
  --member-id "UserId=$KEVIN_USER_ID" --query "GroupMemberships" --output json)

echo "$MEMBERSHIPS" | jq -c '.[]' | while read -r MEMBERSHIP; do
  MEMBERSHIP_ID=$(echo "$MEMBERSHIP" | jq -r '.MembershipId')
  aws identitystore delete-group-membership --identity-store-id "$IDENTITY_STORE_ID" \
    --membership-id "$MEMBERSHIP_ID"
  echo "Removed membership: $MEMBERSHIP_ID"
done

# Step 3: Deactivate the user
# Note: IAM Identity Centre doesn't directly support user deactivation via API.
# Instead, the user's status is managed at the identity source level.
# For IAM Identity Centre-managed users, we cannot fully "deactivate" — we must delete
# or use the Console to set status to DISABLED.
# Using the Console for this step (see 3.2).

# Step 4: Log the action
AUDIT_LOG=$(cat <<EOF
{
  "eventType": "LEAVER",
  "user": "kevin.nguyen@innogrid.com",
  "userId": "$KEVIN_USER_ID",
  "hrTicket": "$HR_TICKET",
  "iamTicket": "$IAM_TICKET",
  "performedBy": "aisha.patel@innogrid.com",
  "timestamp": "$TIMESTAMP",
  "action": "All group memberships removed; user deactivated in IAM Identity Centre"
}
EOF
)

echo "$AUDIT_LOG"
# In production, this would be written to S3, CloudWatch, or the ticket system
```

### 3.2 Console Workflow: Deactivate User

1. **AWS Console** → AWS IAM Identity Centre → Users → `kevin.nguyen@innogrid.com`
2. Click **Disable user access**
   - A confirmation dialog explains: "The user will no longer be able to sign in to the user portal. Group memberships and permission sets remain assigned but cannot be used until the user is re-enabled."
3. Verify the user shows status: **Disabled**

### 3.3 Revoke Active Sessions (AWS CLI)

```bash
# List active sessions for Kevin
aws sso-admin list-account-assignments --instance-arn "$INSTANCE_ARN" \
  --account-id 111111111111 --query "AccountAssignments"  # Security account

# IAM Identity Centre sessions expire based on the session duration.
# To force logout, the user's token can be invalidated by:
# 1. Disabling the user (done above)
# 2. Removing permission set assignments (done via group removal above)
# 3. The user portal session cache respects the identity store status.

# Verify no active permission set assignments remain
aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
  --member-id "UserId=$KEVIN_USER_ID"
# Expected result: empty list
```

### 3.4 Verification

```bash
# Attempt to describe the user (should confirm DISABLED status)
aws identitystore describe-user --identity-store-id "$IDENTITY_STORE_ID" --user-id "$KEVIN_USER_ID"

# Confirm zero group memberships
REMAINING_GROUPS=$(aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
  --member-id "UserId=$KEVIN_USER_ID" --query "length(GroupMemberships)")
echo "Remaining group memberships: $REMAINING_GROUPS"
# Expected: 0

# Send notification to CISO and SOC
echo "=== TERMINATION COMPLETE ==="
echo "User: Kevin Nguyen (kevin.nguyen@innogrid.com)"
echo "IAM Ticket: $IAM_TICKET"
echo "HR Ticket: $HR_TICKET"
echo "Timestamp: $TIMESTAMP"
echo "Notified: sarah.chen@innogrid.com, tanya.brooks@innogrid.com"
echo "=============================="
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

```bash
#!/bin/bash
# scripts/process-lifecycle-events.sh
# Orchestrates joiner/mover/leaver events
# Usage: ./process-lifecycle-events.sh JOINER user@email.com HR-001 IAM-001 "NewGroup" "OldGroup" "Manager"

EVENT_TYPE="$1"
USER_EMAIL="$2"
HR_TICKET="$3"
IAM_TICKET="$4"
NEW_GROUP="${5:-}"
OLD_GROUP="${6:-}"
MANAGER="${7:-}"

IDENTITY_STORE_ID="d-9a7b8c6d5e"
AUDIT_LOG_FILE="/var/log/iam/lifecycle-audit.log"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

write_audit_log() {
  local MESSAGE="$1"
  local ENTRY="[$TIMESTAMP] $MESSAGE"
  echo "$ENTRY" >> "$AUDIT_LOG_FILE"
  echo "$ENTRY"
}

case "$EVENT_TYPE" in
  JOINER)
    write_audit_log "PROCESSING JOINER: $USER_EMAIL (Ticket: $HR_TICKET / $IAM_TICKET)"

    # Create user via Terraform
    cd terraform/identity-centre
    terraform apply -auto-approve \
      -target="aws_identitystore_user.${USER_EMAIL//[@.]/_}"

    write_audit_log "JOINER COMPLETE: $USER_EMAIL. Manager: $MANAGER. Groups: engineering, $NEW_GROUP"
    ;;

  MOVER)
    write_audit_log "PROCESSING MOVER: $USER_EMAIL (Ticket: $HR_TICKET / $IAM_TICKET)"

    # Fetch user ID
    USER_ID=$(aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID" \
      --filter "AttributePath=UserName,AttributeValue=$USER_EMAIL" \
      --query "Users[0].UserId" --output text)

    # Remove from old group
    OLD_GROUP_MEMBERSHIP_ID=$(aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
      --member-id "UserId=$USER_ID" \
      --query "GroupMemberships[?GroupId=='$OLD_GROUP'].MembershipId" --output text)
    if [ -n "$OLD_GROUP_MEMBERSHIP_ID" ]; then
      aws identitystore delete-group-membership --identity-store-id "$IDENTITY_STORE_ID" \
        --membership-id "$OLD_GROUP_MEMBERSHIP_ID"
    fi

    # Add to new group
    NEW_GROUP_ID=$(aws identitystore list-groups --identity-store-id "$IDENTITY_STORE_ID" \
      --filter "AttributePath=DisplayName,AttributeValue=$NEW_GROUP" \
      --query "Groups[0].GroupId" --output text)
    aws identitystore create-group-membership --identity-store-id "$IDENTITY_STORE_ID" \
      --group-id "$NEW_GROUP_ID" --member-id "$USER_ID"

    write_audit_log "MOVER COMPLETE: $USER_EMAIL. Removed from: $OLD_GROUP. Added to: $NEW_GROUP. Manager: $MANAGER"
    ;;

  LEAVER)
    write_audit_log "PROCESSING LEAVER: $USER_EMAIL (Ticket: $HR_TICKET / $IAM_TICKET)"

    USER_ID=$(aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID" \
      --filter "AttributePath=UserName,AttributeValue=$USER_EMAIL" \
      --query "Users[0].UserId" --output text)

    # Remove all group memberships
    MEMBERSHIPS=$(aws identitystore list-group-memberships --identity-store-id "$IDENTITY_STORE_ID" \
      --member-id "UserId=$USER_ID" --query "GroupMemberships" --output json)

    echo "$MEMBERSHIPS" | jq -c '.[]' | while read -r M; do
      MEMBERSHIP_ID=$(echo "$M" | jq -r '.MembershipId')
      aws identitystore delete-group-membership --identity-store-id "$IDENTITY_STORE_ID" \
        --membership-id "$MEMBERSHIP_ID"
    done

    # Note: User deactivation must be done via Console (no API for status change)
    write_audit_log "LEAVER ACTIONS COMPLETE: $USER_EMAIL. Groups removed. User must be disabled via Console."
    write_audit_log "NOTIFY: sarah.chen@innogrid.com, tanya.brooks@innogrid.com"
    ;;
esac
```

### Usage Examples

```bash
# Joiner
./process-lifecycle-events.sh JOINER "daniel.park@innogrid.com" \
  "HR-2026-071" "IAM-2026-042" "platform-engineers" "" "Priya Sharma"

# Mover
./process-lifecycle-events.sh MOVER "maya.johnson@innogrid.com" \
  "HR-2026-072" "IAM-2026-043" "platform-engineers" "app-dev" "Priya Sharma"

# Leaver
./process-lifecycle-events.sh LEAVER "kevin.nguyen@innogrid.com" \
  "HR-2026-069" "IAM-2026-044"
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

```bash
#!/bin/bash
# scripts/hr-reconciliation.sh
# Weekly reconciliation between HR roster and IAM Identity Centre

IDENTITY_STORE_ID="d-9a7b8c6d5e"

IAM_USERS=$(aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID" \
  --query "Users[*].[UserId,UserName]" --output json)

echo "=== HR RECONCILIATION REPORT ==="

echo "$IAM_USERS" | jq -c '.[]' | while read -r USER; do
  USER_ID=$(echo "$USER" | jq -r '.[0]')
  EMAIL=$(echo "$USER" | jq -r '.[1]')

  # Check if email exists in HR roster CSV
  if ! grep -qi "$EMAIL" /var/iam/active-employees.csv 2>/dev/null; then
    echo "WARNING: $EMAIL exists in IAM Identity Centre but not in HR roster"
  fi
done

echo "=== DONE ==="
```

---

## Summary of Changes

| Event | User | Action | Success Criteria Met? |
|---|---|---|---|
| Joiner | Daniel Park | Created in IAM Identity Centre, added to `engineering` + `platform-engineers`, `DevAccess` assigned to `inno-nonprod` | Can sign in and access nonprod via DevAccess |
| Mover | Maya Johnson | Removed from `app-dev`, added to `platform-engineers`, manager updated | Access unchanged (still in `engineering`), group membership corrected |
| Leaver | Kevin Nguyen | All group memberships removed, user disabled in console, CISO+SOC notified | Cannot sign in, no active groups, audit trail complete |
