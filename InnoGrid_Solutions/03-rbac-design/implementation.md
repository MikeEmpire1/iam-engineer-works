# Scenario 3: RBAC Design — Implementation

## Prerequisites

- Terraform 1.5+ with AWS provider configured for Management account
- AWS IAM Identity Centre enabled with instance ARN
- Existing ad-hoc groups documented for migration tracking
- Jira project for JIT approval workflow

---

## 1. Terraform RBAC Module

```hcl
# terraform/modules/rbac/main.tf
# Reusable RBAC module for IAM Identity Centre

variable "identity_store_id" {
  type = string
}

variable "instance_arn" {
  type = string
}

variable "functions" {
  description = "Map of job functions to their roles"
  type = map(map(object({
    permission_sets = list(string)
    accounts        = list(string)
  })))
  default = {}
}

# Permission set resources
# These are defined centrally and referenced by groups

resource "aws_ssoadmin_permission_set" "this" {
  for_each = var.permission_sets

  instance_arn = var.instance_arn
  name         = each.key
  description  = each.value.description
  session_duration = each.value.session_duration
  relay_state      = try(each.value.relay_state, null)
}

resource "aws_ssoadmin_permission_set_inline_policy" "this" {
  for_each = {
    for k, v in var.permission_sets : k => v if v.inline_policy != null
  }

  instance_arn       = var.instance_arn
  permission_set_arn = aws_ssoadmin_permission_set.this[each.key].arn
  inline_policy      = jsonencode(each.value.inline_policy)
}

resource "aws_ssoadmin_managed_policy_attachment" "this" {
  for_each = local.managed_policy_attachments

  instance_arn       = var.instance_arn
  managed_policy_arn = each.value.policy_arn
  permission_set_arn = each.value.ps_arn
}

# Groups
resource "aws_identitystore_group" "this" {
  for_each = var.groups

  identity_store_id = var.identity_store_id
  display_name      = each.key
  description       = each.value.description
}

# Group memberships are managed externally (lifecycle events),
# but group-to-permission-set assignments are defined here

resource "aws_ssoadmin_account_assignment" "this" {
  for_each = local.assignments

  instance_arn       = var.instance_arn
  permission_set_arn = each.value.ps_arn
  principal_type     = "GROUP"
  principal_id       = aws_identitystore_group.this[each.value.group].group_id
  target_id          = each.value.account_id
  target_type        = "AWS_ACCOUNT"
}
```

---

## 2. RBAC Configuration

```hcl
# terraform/environments/prod/rbac-config.tf

module "rbac" {
  source = "../../modules/rbac"

  identity_store_id = "d-9a7b8c6d5e"
  instance_arn      = "arn:aws:sso:::instance/ssoins-1234567890abcdef"

  permission_sets = {
    "Nonprod-Dev" = {
      description      = "Developer access to Nonproduction account"
      session_duration = "PT8H"
      relay_state      = "https://console.aws.amazon.com/ec2"
      inline_policy = {
        Version = "2012-10-17"
        Statement = [
          {
            Effect = "Allow"
            Action = [
              "ec2:StartInstances",
              "ec2:StopInstances",
              "ec2:DescribeInstances",
              "ssm:StartSession"
            ]
            Resource = ["*"]
          },
          {
            Effect = "Allow"
            Action = ["s3:GetObject", "s3:ListBucket"]
            Resource = [
              "arn:aws:s3:::inno-nonprod-dev-*",
              "arn:aws:s3:::inno-nonprod-dev-*/*"
            ]
          }
        ]
      }
    }
    "Prod-ReadOnly" = {
      description      = "Read-only access to Production account"
      session_duration = "PT4H"
      inline_policy = {
        Version = "2012-10-17"
        Statement = [
          {
            Effect = "Deny"
            Action = [
              "iam:Create*",
              "iam:Update*",
              "iam:Delete*",
              "ec2:RunInstances",
              "ec2:StartInstances",
              "ec2:StopInstances",
              "ec2:TerminateInstances",
              "s3:PutObject",
              "s3:DeleteObject",
              "rds:Create*",
              "rds:Modify*",
              "rds:Delete*",
              "lambda:Create*",
              "lambda:Update*",
              "lambda:Delete*",
              "sns:Publish"
            ]
            Resource = ["*"]
          }
        ]
      }
    }
    "Security-ReadOnly" = {
      description      = "Read-only access to Security account"
      session_duration = "PT4H"
      inline_policy    = null
    }
    "Security-Ops" = {
      description      = "Operational access to Security account"
      session_duration = "PT4H"
      inline_policy = {
        Version = "2012-10-17"
        Statement = [
          {
            Effect = "Allow"
            Action = [
              "guardduty:*",
              "securityhub:*",
              "cloudtrail:*",
              "config:*",
              "s3:Get*",
              "s3:List*"
            ]
            Resource = ["*"]
          }
        ]
      }
    }
    "Prod-Admin-JIT" = {
      description      = "JIT production admin access (auto-expires 1 hour)"
      session_duration = "PT1H"
      relay_state      = "https://console.aws.amazon.com/cloudwatch/home"
      inline_policy    = null
    }
  }

  groups = {
    "platform-engineering-engineer" = {
      description = "Platform engineers with dev access"
    }
    "platform-engineering-senior" = {
      description = "Senior platform engineers with security read access"
    }
    "platform-engineering-lead" = {
      description = "Platform engineering leads with expanded access"
    }
    "app-dev-engineer" = {
      description = "Application developers"
    }
    "app-dev-senior" = {
      description = "Senior application developers"
    }
    "qa-engineer" = {
      description = "QA engineers"
    }
    "iam-admin" = {
      description = "IAM team members"
    }
    "soc-analyst" = {
      description = "SOC analysts"
    }
    "soc-support" = {
      description = "SOC support"
    }
    "service-desk-support" = {
      description = "Service Desk technicians"
    }
    "finance-reader" = {
      description = "Finance team (read-only)"
    }
    "hr-reader" = {
      description = "HR team (read-only)"
    }
    "legal-reader" = {
      description = "Legal team (read-only)"
    }
    "marketing-reader" = {
      description = "Marketing team (read-only)"
    }
    "operations-reader" = {
      description = "Operations team (read-only)"
    }
    "executive-reader" = {
      description = "Executive team (read-only all accounts)"
    }
  }

  # Account assignments: group → permission set → account
  group_assignments = {
    platform-engineering-engineer = [
      { permission_set = "Nonprod-Dev", account_id = "123456789012" },
      { permission_set = "Prod-ReadOnly", account_id = "777788889999" },
    ]
    platform-engineering-senior = [
      { permission_set = "Nonprod-Dev", account_id = "123456789012" },
      { permission_set = "Prod-ReadOnly", account_id = "777788889999" },
      { permission_set = "Security-ReadOnly", account_id = "444455556666" },
    ]
    iam-admin = [
      { permission_set = "Nonprod-Dev", account_id = "123456789012" },
      { permission_set = "Security-Ops", account_id = "444455556666" },
      { permission_set = "Prod-ReadOnly", account_id = "777788889999" },
    ]
    finance-reader = [
      { permission_set = "Prod-ReadOnly", account_id = "777788889999" },
    ]
    executive-reader = [
      { permission_set = "Prod-ReadOnly", account_id = "777788889999" },
      { permission_set = "Security-ReadOnly", account_id = "444455556666" },
      { permission_set = "Nonprod-Dev", account_id = "123456789012" },
    ]
    # ... (all other groups follow same pattern)
  }
}
```

---

## 3. Apply RBAC

```bash
cd terraform/environments/prod
terraform init
terraform plan -out=rbac.tfplan
terraform apply rbac.tfplan
```

---

## 4. JIT Elevation Setup

### Step 1: Create the JIT Permission Set (already in Terraform above)

### Step 2: JIT Assignment Lambda

```python
# lambdas/jit_elevation/app.py
"""Self-service JIT elevation handler triggered by IAM Identity Centre events."""

import boto3
import os
import json
from datetime import datetime, timezone

sso_admin = boto3.client("sso-admin")
identity_store = boto3.client("identitystore")

instance_arn = os.environ["INSTANCE_ARN"]
identity_store_id = os.environ["IDENTITY_STORE_ID"]
jit_ps_arn = os.environ["JIT_PERMISSION_SET_ARN"]
prod_account_id = os.environ["PROD_ACCOUNT_ID"]

def lambda_handler(event, context):
    """Handle JIT elevation requests from the access portal or Jira webhook."""
    body = json.loads(event.get("body", "{}"))
    user_email = body["user_email"]
    action = body.get("action", "assume")  # assume or release
    duration = int(body.get("duration_hours", 1))

    # Look up user
    users = identity_store.list_users(
        IdentityStoreId=identity_store_id,
        Filters=[{"AttributePath": "UserName", "AttributeValue": user_email}]
    )
    if not users.get("Users"):
        return {"statusCode": 404, "body": json.dumps({"error": "User not found"})}

    user_id = users["Users"][0]["UserId"]

    if action == "assume":
        # Create a temporary account assignment
        # Note: In real implementation, this would use AWS SSO
        # JIT features or a custom session manager
        sso_admin.create_account_assignment(
            InstanceArn=instance_arn,
            TargetId=prod_account_id,
            TargetType="AWS_ACCOUNT",
            PermissionSetArn=jit_ps_arn,
            PrincipalType="USER",
            PrincipalId=user_id
        )

        # Schedule auto-revocation (CloudWatch Events + Lambda)
        # In production, this would create a one-time CloudWatch event
        # to revoke access after the duration expires

        return {
            "statusCode": 200,
            "body": json.dumps({
                "message": f"JIT elevation granted for {duration} hour(s)",
                "expires_at": datetime.now(timezone.utc).isoformat()
            })
        }

    elif action == "release":
        # Remove the temporary assignment
        sso_admin.delete_account_assignment(
            InstanceArn=instance_arn,
            TargetId=prod_account_id,
            TargetType="AWS_ACCOUNT",
            PermissionSetArn=jit_ps_arn,
            PrincipalType="USER",
            PrincipalId=user_id
        )
        return {"statusCode": 200, "body": json.dumps({"message": "JIT elevation released"})}

    return {"statusCode": 400, "body": json.dumps({"error": "Invalid action"})}
```

### Step 3: JIT Auto-Expiry CloudWatch Event Rule

```hcl
# terraform/modules/jit/cloudwatch.tf

resource "aws_cloudwatch_event_rule" "jit_timeout" {
  name                = "jit-elevation-timeout"
  description         = "Auto-revoke JIT elevations after expiry"
  schedule_expression = "rate(5 minutes)"
}

resource "aws_cloudwatch_event_target" "jit_revoker" {
  rule      = aws_cloudwatch_event_rule.jit_timeout.name
  target_id = "RevokeExpiredJit"
  arn       = aws_lambda_function.jit_revoker.arn
}
```

---

## 5. Migration Script

### PowerShell: Migrate Users to New Groups

```bash
# scripts/migrate-to-rbac.sh
identityStoreId="d-9a7b8c6d5e"

declare -A migration
migration["alex.rivera@innogrid.com"]="platform-engineering-senior"
migration["sam.green@innogrid.com"]="platform-engineering-engineer"
migration["daniel.park@innogrid.com"]="platform-engineering-engineer"
migration["maya.johnson@innogrid.com"]="platform-engineering-engineer"
migration["ethan.brown@innogrid.com"]="app-dev-engineer"
migration["chloe.wilson@innogrid.com"]="qa-engineer"
migration["aisha.patel@innogrid.com"]="iam-admin"
migration["miguel.torres@innogrid.com"]="iam-admin"
migration["jake.hoffman@innogrid.com"]="soc-analyst"
migration["olivia.reed@innogrid.com"]="soc-analyst"
migration["emily.zhao@innogrid.com"]="service-desk-support"
migration["amanda.foster@innogrid.com"]="hr-reader"
migration["jordan.bell@innogrid.com"]="hr-reader"
migration["diana.cruz@innogrid.com"]="finance-reader"
migration["sarah.huang@innogrid.com"]="legal-reader"
migration["nina.patel@innogrid.com"]="marketing-reader"
migration["chris.evans@innogrid.com"]="marketing-reader"
migration["tom.watson@innogrid.com"]="marketing-reader"
migration["ben.schneider@innogrid.com"]="operations-reader"
migration["maria.gomez@innogrid.com"]="operations-reader"
migration["priya.sharma@innogrid.com"]="platform-engineering-lead"
migration["derek.jones@innogrid.com"]="app-dev-senior"
migration["lisa.kim@innogrid.com"]="qa-engineer"
migration["ryan.mitchell@innogrid.com"]="iam-admin"
migration["tanya.brooks@innogrid.com"]="soc-analyst"
migration["carlos.mendez@innogrid.com"]="service-desk-support"
migration["james.okafo@innogrid.com"]="platform-engineering-lead"
migration["david.park@innogrid.com"]="executive-reader"
migration["elena.vasquez@innogrid.com"]="executive-reader"
migration["mark.tanaka@innogrid.com"]="executive-reader"
migration["sarah.chen@innogrid.com"]="executive-reader"

oldGroups=("Engineering-Team" "DevAccess-Group" "app-dev" "platform-engineers"
           "qa-engineers" "iam-engineers" "soc-team" "helpdesk"
           "finance-team" "hr-team" "legal-team" "marketing-team"
           "operations-team" "exec-team" "Alex-Admin" "old-group-2024")

echo "=== PHASE 2: Adding users to new RBAC groups ==="
for email in "${!migration[@]}"; do
    userId=$(aws identitystore list-users --identity-store-id "$identityStoreId" \
        --filter AttributePath=UserName,AttributeValue="$email" \
        --query "Users[0].UserId" --output text)

    groupName="${migration[$email]}"
    groupId=$(aws identitystore list-groups --identity-store-id "$identityStoreId" \
        --filter AttributePath=DisplayName,AttributeValue="$groupName" \
        --query "Groups[0].GroupId" --output text)

    aws identitystore create-group-membership --identity-store-id "$identityStoreId" \
        --group-id "$groupId" --member-id "UserId=$userId"
    echo "  + $email → $groupName"
done

echo "=== PHASE 4: Removing users from old groups ==="
for email in "${!migration[@]}"; do
    userId=$(aws identitystore list-users --identity-store-id "$identityStoreId" \
        --filter AttributePath=UserName,AttributeValue="$email" \
        --query "Users[0].UserId" --output text)

    for oldGroup in "${oldGroups[@]}"; do
        groupId=$(aws identitystore list-groups --identity-store-id "$identityStoreId" \
            --filter AttributePath=DisplayName,AttributeValue="$oldGroup" \
            --query "Groups[0].GroupId" --output text)
        if [ -z "$groupId" ]; then continue; fi

        membershipId=$(aws identitystore list-group-memberships --identity-store-id "$identityStoreId" \
            --group-id "$groupId" --query "GroupMemberships[?MemberId.UserId=='$userId'].MembershipId" --output text)
        if [ -n "$membershipId" ]; then
            aws identitystore delete-group-membership --identity-store-id "$identityStoreId" \
                --membership-id "$membershipId"
            echo "  - $email → $oldGroup (removed)"
        fi
    done
done

echo "=== PHASE 5: Cleaning up orphaned groups ==="
for oldGroup in "${oldGroups[@]}"; do
    groupId=$(aws identitystore list-groups --identity-store-id "$identityStoreId" \
        --filter AttributePath=DisplayName,AttributeValue="$oldGroup" \
        --query "Groups[0].GroupId" --output text)
    if [ -n "$groupId" ]; then
        aws identitystore list-group-memberships --identity-store-id "$identityStoreId" \
            --group-id "$groupId" --query "length(GroupMemberships)"
        # If zero members, safe to delete
        aws identitystore delete-group --identity-store-id "$identityStoreId" --group-id "$groupId"
        echo "  DELETED: $oldGroup"
    fi
done

echo "=== MIGRATION COMPLETE ==="
```

---

## 6. Verification

```bash
# Verify Alex Rivera has correct RBAC group memberships
aws identitystore list-group-memberships --identity-store-id d-9a7b8c6d5e \
  --member-id UserId="<alex-user-id>"
# Expected: platform-engineering-senior only (no old groups)

# Verify Finance reader can access Prod
aws sso-admin list-account-assignments --instance-arn "arn:aws:sso:::instance/ssoins-1234567890abcdef" \
  --account-id 777788889999 --principal-type GROUP --principal-id "<finance-reader-group-id>"
# Expected: Prod-ReadOnly

# Verify no old groups remain
aws identitystore list-groups --identity-store-id d-9a7b8c6d5e | jq
# Expected: Only the 16 RBAC groups
```

---

## 7. Console Workflow

### Create a Permission Set

1. **AWS Console** → IAM Identity Centre → Permission sets → **Create permission set**
2. Select **Custom permission set**
3. Name: `Nonprod-Dev`
4. Session duration: 8 hours
5. Relay state: `https://console.aws.amazon.com/ec2`
6. Attach managed policies: `ReadOnlyAccess`
7. Add inline policy (see Terraform above)
8. Tags: `RBAC: true`

### Assign a Group to an Account

1. **AWS Console** → IAM Identity Centre → AWS accounts → `inno-nonprod`
2. **Assign users/groups** → Select `platform-engineering-engineer`
3. Select permission set: `Nonprod-Dev`
4. Confirm

---

## Summary

| Old State | New State |
|---|---|
| 50+ ad-hoc groups | 16 standardised RBAC groups |
| 8+ duplicate permission sets | 5 distinct permission set tiers |
| Standing prod admin access | JIT elevation with 1-hour auto-expiry |
| No naming convention | `<function>-<role>` format |
| Orphaned groups from 2024 | Cleaned up |
| Manual group management | Terraform modules + CI/CD |
