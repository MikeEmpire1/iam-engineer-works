# Scenario 2: Access Certification — Implementation

## Prerequisites

- AWS CLI configured with Management account access
- Python 3.9+ with `boto3` and `pandas` installed
- Entra ID admin access for Access Reviews
- HR roster CSV exported from Workday

---

## 1. Generate Review Manifest

### Step 1: Export Current IAM Identity Centre Assignments

```python
# scripts/generate_review_manifest.py
"""Queries IAM Identity Centre and generates a CSV review manifest."""

import boto3
import csv
import json

identity_store_id = "d-9a7b8c6d5e"
instance_arn = "arn:aws:sso:::instance/ssoins-1234567890abcdef"
identity_client = boto3.client("identitystore")
sso_admin = boto3.client("sso-admin")

# Fetch all users
users = {}
paginator = identity_client.get_paginator("list_users")
for page in paginator.paginate(IdentityStoreId=identity_store_id):
    for u in page["Users"]:
        users[u["UserId"]] = {
            "email": u.get("UserName", ""),
            "name": u.get("DisplayName", ""),
        }

# Fetch all groups
groups = {}
paginator = identity_client.get_paginator("list_groups")
for page in paginator.paginate(IdentityStoreId=identity_store_id):
    for g in page["Groups"]:
        groups[g["GroupId"]] = g["DisplayName"]

# Fetch group memberships for each user
user_groups = {}
for uid in users:
    user_groups[uid] = []
    paginator = identity_client.get_paginator("list_group_memberships")
    for page in paginator.paginate(IdentityStoreId=identity_store_id, MemberId={"UserId": uid}):
        for m in page["GroupMemberships"]:
            user_groups[uid].append(m["GroupId"])

# Fetch account assignments for each group
account_assignments = {}
for gid in groups:
    account_assignments[gid] = []
    paginator = sso_admin.get_paginator("list_account_assignments")
    for page in paginator.paginate(
        InstanceArn=instance_arn,
        AccountId="123456789012",  # inno-nonprod
        PermissionSetArn="arn:aws:sso:::permissionSet/ssoins-1234567890abcdef/ps-devaccess",
    ):
        for a in page["AccountAssignments"]:
            account_assignments[gid].append({
                "account_id": "123456789012",
                "account_name": "inno-nonprod",
                "permission_set": "DevAccess",
            })

# Build CSV
rows = []
manager_map = {
    "alex.rivera@innogrid.com": "priya.sharma@innogrid.com",
    "sam.green@innogrid.com": "priya.sharma@innogrid.com",
    "daniel.park@innogrid.com": "priya.sharma@innogrid.com",
    "ethan.brown@innogrid.com": "derek.jones@innogrid.com",
    "chloe.wilson@innogrid.com": "lisa.kim@innogrid.com",
    "aisha.patel@innogrid.com": "ryan.mitchell@innogrid.com",
    "miguel.torres@innogrid.com": "ryan.mitchell@innogrid.com",
    "jake.hoffman@innogrid.com": "tanya.brooks@innogrid.com",
    "olivia.reed@innogrid.com": "tanya.brooks@innogrid.com",
    "emily.zhao@innogrid.com": "carlos.mendez@innogrid.com",
    "amanda.foster@innogrid.com": "rebecca.torres@innogrid.com",
    "jordan.bell@innogrid.com": "rebecca.torres@innogrid.com",
    "diana.cruz@innogrid.com": "nathan.cole@innogrid.com",
    "grace.kim@innogrid.com": "nathan.cole@innogrid.com",
    "sarah.huang@innogrid.com": "peter.griffin@innogrid.com",
    "nina.patel@innogrid.com": "rachel.adams@innogrid.com",
    "chris.evans@innogrid.com": "rachel.adams@innogrid.com",
    "tom.watson@innogrid.com": "rachel.adams@innogrid.com",
    "ben.schneider@innogrid.com": "angela.wright@innogrid.com",
    "maria.gomez@innogrid.com": "angela.wright@innogrid.com",
}

manager_names = {
    "priya.sharma@innogrid.com": "Priya Sharma",
    "derek.jones@innogrid.com": "Derek Jones",
    "lisa.kim@innogrid.com": "Lisa Kim",
    "ryan.mitchell@innogrid.com": "Ryan Mitchell",
    "tanya.brooks@innogrid.com": "Tanya Brooks",
    "carlos.mendez@innogrid.com": "Carlos Mendez",
    "rebecca.torres@innogrid.com": "Rebecca Torres",
    "nathan.cole@innogrid.com": "Nathan Cole",
    "peter.griffin@innogrid.com": "Peter Griffin",
    "rachel.adams@innogrid.com": "Rachel Adams",
    "angela.wright@innogrid.com": "Angela Wright",
}

with open("q3-2026-review-manifest.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow([
        "user_email", "user_name", "department", "manager_email",
        "manager_name", "group", "permission_set", "account_id",
        "account_name", "decision", "notes"
    ])
    for uid, info in users.items():
        email = info["email"]
        manager_email = manager_map.get(email, "")
        mgr_name = manager_names.get(manager_email, "")
        for gid in user_groups.get(uid, []):
            for assignment in account_assignments.get(gid, []):
                writer.writerow([
                    email, info["name"], "Engineering",
                    manager_email, mgr_name,
                    groups.get(gid, ""),
                    assignment["permission_set"],
                    assignment["account_id"],
                    assignment["account_name"],
                    "",  # decision — to be filled by manager
                    "",  # notes
                ])

print("Review manifest generated: q3-2026-review-manifest.csv")
```

**Run:**
```powershell
python scripts/generate_review_manifest.py
```

---

## 2. Distribute Review Packages

### PowerShell: Send Review Invitations

```powershell
# scripts/send-review-invitations.ps1
$manifest = Import-Csv "q3-2026-review-manifest.csv"
$reviewers = $manifest | Group-Object manager_email

foreach ($reviewer in $reviewers) {
    $mgrEmail = $reviewer.Name
    $mgrName = $reviewer.Group[0].manager_name
    $reportCount = ($reviewer.Group | Select-Object user_email -Unique).Count
    $reportList = ($reviewer.Group | Select-Object user_name -Unique).user_name -join ", "

    $body = @"
Hi $mgrName,

The Q3 2026 access review campaign is now open. You have $reportCount direct reports to review:

Reports: $reportList

Please review their access and submit your decisions by July 14, 2026.

Instructions:
1. Open the attached CSV: q3-2026-review-manifest.csv
2. Filter by your email in the manager_email column
3. For each row, fill in the decision column (APPROVE / MODIFY / REVOKE / SUSPEND)
4. Add any notes in the notes column
5. Return the completed CSV to the IAM team

If all access looks correct, you can simply reply "APPROVE ALL" to this email.

Thank you,
IAM Team
"@

    Write-Host "Sending review invitation to $mgrEmail ($mgrName)"
    Write-Host "Reports: $reportList"
    Write-Host "---"
    # In production, this would use Send-MailMessage or an API
}
```

---

## 3. Process Manager Decisions

### Python: Process Review Decisions

```python
# scripts/process_review_decisions.py
"""Takes the completed review CSV and applies decisions to IAM Identity Centre."""

import boto3
import csv

identity_store_id = "d-9a7b8c6d5e"
identity_client = boto3.client("identitystore")

# Expected decisions for this campaign
decisions = {
    "miguel.torres@innogrid.com": {
        "groups_to_remove": ["sandbox-users"],  # Remove SandboxAccess
        "action": "MODIFY",
        "notes": "Remove Sandbox access - no longer needed for project"
    },
    "grace.kim@innogrid.com": {
        "action": "SUSPEND",
        "notes": "On parental leave until 2026-08-01"
    },
    "chris.evans@innogrid.com": {
        "groups_to_remove": ["marketing-dev"],  # Remove DevAccess on Nonproduction
        "action": "MODIFY",
        "notes": "Remove Nonproduction access - no longer needs dev environment"
    },
}

def get_user_id(email):
    response = identity_client.list_users(
        IdentityStoreId=identity_store_id,
        Filters=[{"AttributePath": "UserName", "AttributeValue": email}]
    )
    users = response.get("Users", [])
    return users[0]["UserId"] if users else None

def get_group_id(display_name):
    response = identity_client.list_groups(
        IdentityStoreId=identity_store_id,
        Filters=[{"AttributePath": "DisplayName", "AttributeValue": display_name}]
    )
    groups = response.get("Groups", [])
    return groups[0]["GroupId"] if groups else None

def remove_group_membership(user_id, group_id):
    memberships = identity_client.list_group_memberships(
        IdentityStoreId=identity_store_id,
        MemberId={"UserId": user_id}
    )
    for m in memberships.get("GroupMemberships", []):
        if m["GroupId"] == group_id:
            identity_client.delete_group_membership(
                IdentityStoreId=identity_store_id,
                MembershipId=m["MembershipId"]
            )
            print(f"  Removed from group: {m['GroupId']}")
            return

# Process modifications
for email, decision in decisions.items():
    print(f"\nProcessing: {email} ({decision['action']})")
    user_id = get_user_id(email)
    if not user_id:
        print(f"  ERROR: User not found")
        continue

    if decision["action"] == "MODIFY" and "groups_to_remove" in decision:
        for group_name in decision["groups_to_remove"]:
            group_id = get_group_id(group_name)
            if group_id:
                remove_group_membership(user_id, group_id)
                print(f"  Removed from {group_name}")

    elif decision["action"] == "SUSPEND":
        print(f"  SUSPEND: {email} - {decision['notes']}")
        print(f"  Action: Remove all group memberships (manual confirmation required)")
        # For suspension, remove all group memberships
        memberships = identity_client.list_group_memberships(
            IdentityStoreId=identity_store_id,
            MemberId={"UserId": user_id}
        )
        for m in memberships.get("GroupMemberships", []):
            identity_client.delete_group_membership(
                IdentityStoreId=identity_store_id,
                MembershipId=m["MembershipId"]
            )
            print(f"  Removed from group: {m['GroupId']}")

print("\nReview processing complete.")
```

---

## 4. Auto-Revocation of Unrecertified Access

### PowerShell: Revoke Unrecertified Users

```powershell
# scripts/revoke-unrecertified.ps1
# Run after July 14 deadline
$manifest = Import-Csv "q3-2026-review-manifest.csv"
$unreviewed = $manifest | Where-Object { [string]::IsNullOrWhiteSpace($_.decision) }

if ($unreviewed.Count -eq 0) {
    Write-Host "All access has been recertified. No auto-revocation needed."
    exit 0
}

Write-Host "Found $($unreviewed.Count) unrecertified assignments. Revoking..."
foreach ($row in $unreviewed) {
    Write-Host "  REVOKING: $($row.user_email) - $($row.group) on $($row.account_name)"
    # In production, this would remove the user from the group via AWS CLI
    # aws identitystore delete-group-membership ...
}
```

---

## 5. Restore Leave Employee (Grace Kim)

### PowerShell: Restore Grace Kim's Access

```powershell
# scripts/restore-leave-user.ps1
# Run on 2026-08-01 when Grace Kim returns from leave
$identityStoreId = "d-9a7b8c6d5e"

$graceUserId = aws identitystore list-users --identity-store-id $identityStoreId `
  --filter AttributePath=UserName,AttributeValue=grace.kim@innogrid.com `
  --query "Users[0].UserId" --output text

# Get the finance group ID
$financeGroupId = aws identitystore list-groups --identity-store-id $identityStoreId `
  --filter AttributePath=DisplayName,AttributeValue=finance `
  --query "Groups[0].GroupId" --output text

# Re-add Grace to the finance group
aws identitystore create-group-membership --identity-store-id $identityStoreId `
  --group-id $financeGroupId --member-id $graceUserId

Write-Host "Grace Kim restored to finance group. Access reactivated."
```

---

## 6. Generate Compliance Report

### Python: Generate Audit Report

```python
# scripts/generate_compliance_report.py

import json
from datetime import datetime

report = {
    "campaign": "Q3-2026",
    "organization": "InnoGrid Solutions",
    "period": {
        "start": "2026-07-01",
        "end": "2026-07-14"
    },
    "scope": "All IAM Identity Centre users with AWS account access",
    "total_users": 19,
    "total_assignments": 24,
    "decisions": {
        "approved": 18,
        "modified": 2,
        "suspended": 1,
        "revoked": 0,
        "auto_revoked": 0
    },
    "completion_rate": "100%",
    "reviewers": [
        {
            "name": "Priya Sharma",
            "email": "priya.sharma@innogrid.com",
            "reviewed": ["Alex Rivera", "Sam Green", "Daniel Park"],
            "completed_at": "2026-07-10T14:30:00Z"
        },
        {
            "name": "Derek Jones",
            "email": "derek.jones@innogrid.com",
            "reviewed": ["Ethan Brown"],
            "completed_at": "2026-07-09T09:15:00Z"
        },
        {
            "name": "Lisa Kim",
            "email": "lisa.kim@innogrid.com",
            "reviewed": ["Chloe Wilson"],
            "completed_at": "2026-07-08T11:00:00Z"
        },
        {
            "name": "Ryan Mitchell",
            "email": "ryan.mitchell@innogrid.com",
            "reviewed": ["Aisha Patel", "Miguel Torres"],
            "completed_at": "2026-07-11T16:45:00Z"
        },
        {
            "name": "Tanya Brooks",
            "email": "tanya.brooks@innogrid.com",
            "reviewed": ["Jake Hoffman", "Olivia Reed"],
            "completed_at": "2026-07-07T10:30:00Z"
        },
        {
            "name": "Carlos Mendez",
            "email": "carlos.mendez@innogrid.com",
            "reviewed": ["Emily Zhao"],
            "completed_at": "2026-07-07T08:00:00Z"
        },
        {
            "name": "Rebecca Torres",
            "email": "rebecca.torres@innogrid.com",
            "reviewed": ["Amanda Foster", "Jordan Bell"],
            "completed_at": "2026-07-12T13:20:00Z"
        },
        {
            "name": "Nathan Cole",
            "email": "nathan.cole@innogrid.com",
            "reviewed": ["Diana Cruz", "Grace Kim"],
            "completed_at": "2026-07-10T09:00:00Z"
        },
        {
            "name": "Peter Griffin",
            "email": "peter.griffin@innogrid.com",
            "reviewed": ["Sarah Huang"],
            "completed_at": "2026-07-08T15:30:00Z"
        },
        {
            "name": "Rachel Adams",
            "email": "rachel.adams@innogrid.com",
            "reviewed": ["Nina Patel", "Chris Evans", "Tom Watson"],
            "completed_at": "2026-07-11T11:15:00Z"
        },
        {
            "name": "Angela Wright",
            "email": "angela.wright@innogrid.com",
            "reviewed": ["Ben Schneider", "Maria Gomez"],
            "completed_at": "2026-07-09T14:00:00Z"
        }
    ],
    "changes_applied": [
        {
            "user": "Miguel Torres",
            "change": "Removed from sandbox-users group",
            "reason": "No longer needs Sandbox access",
            "applied_at": "2026-07-11T17:00:00Z"
        },
        {
            "user": "Chris Evans",
            "change": "Removed from marketing-dev group",
            "reason": "No longer needs Nonproduction access",
            "applied_at": "2026-07-11T11:30:00Z"
        },
        {
            "user": "Grace Kim",
            "change": "Suspended from all groups",
            "reason": "Parental leave until 2026-08-01",
            "applied_at": "2026-07-10T09:15:00Z"
        }
    ],
    "controls_mapped": [
        "Cyber Essentials Plus - Logical access",
        "Cyber Essentials Plus - Periodic access review",
        "ISO 27001 A.9.2.5 - Review of user access rights",
        "ISO 27001 A.9.2.6 - Removal of access rights"
    ],
    "generated_at": datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ"),
    "generated_by": "IAM Team (Aisha Patel)"
}

with open("q3-2026-compliance-report.json", "w") as f:
    json.dump(report, f, indent=2)

print("Compliance report generated: q3-2026-compliance-report.json")
```

---

## 7. CloudTrail Audit Query

```sql
-- Athena: Verify review-related changes in CloudTrail
SELECT
    eventtime,
    eventname,
    JSON_EXTRACT_SCALAR(requestparameters, '$.GroupId') AS group_id,
    JSON_EXTRACT_SCALAR(responseelements, '$.MembershipId') AS membership_id,
    useridentity.arn AS actor
FROM cloudtrail_logs
WHERE
    eventname IN (
        'CreateGroupMembership',
        'DeleteGroupMembership',
        'UpdateUser'
    )
    AND eventtime >= '2026-07-01T00:00:00Z'
    AND eventtime <= '2026-07-15T00:00:00Z'
ORDER BY eventtime;
```

---

## 8. Console Workflow (Manual Alternative)

### Create an Entra ID Access Review (Corporate Users)

1. **Azure Portal** → Entra ID → Identity Governance → Access Reviews → **New access review**
2. Select: **Teams + Groups** → Choose `finance`, `hr`, `legal`, `exec`, `marketing`, `operations` groups
3. Scope: All users
4. Reviewers: Group owners (managers)
5. Schedule: One-time (July 1–14)
6. Auto-apply: Disabled (IAM team reviews results first)
7. On completion: Export results → merge with IAM Identity Centre report

### Apply Group Changes in IAM Identity Centre Console

1. **AWS Console** → IAM Identity Centre → Groups
2. For **Miguel Torres**: Open `iam-engineers` group → remove user; open `sandbox-users` → remove user
3. For **Chris Evans**: Open `marketing-dev` group → remove user
4. For **Grace Kim**: Open each group she belongs to → remove user from all
5. Verify changes in user profile → Groups tab

---

## Summary

| Action | User | Change | Method |
|---|---|---|---|
| Modify | Miguel Torres | Removed from `sandbox-users` | AWS CLI / Script |
| Suspend | Grace Kim | Removed from all groups | AWS CLI / Script |
| Modify | Chris Evans | Removed from `marketing-dev` | AWS CLI / Script |
| Approve | All others | No change | Recertified in manifest |
| Restore | Grace Kim | Added back to `finance` | AWS CLI (Aug 1) |
