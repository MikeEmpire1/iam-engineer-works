# Scenario 6: IAM Incident Response — Implementation

## Prerequisites

- AWS CLI configured with Security account access (read-only for investigation, write for containment)
- GuardDuty, Security Hub, and CloudTrail enabled
- EventBridge rule configured to send HIGH severity findings to PagerDuty
- Access to Security account's break-glass credentials (for containment actions)
- Jira for tracking the incident ticket

---

## 1. Acknowledge & Classify

### SOC Response (Tanya Brooks)

```powershell
# Timestamp: 2026-07-15 14:33 UTC
Write-Host "=== INCIDENT ACKNOWLEDGED ==="
Write-Host "Finding: UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS"
Write-Host "Severity: HIGH (7.0)"
Write-Host "User: svc-cicd-deploy (inno-nonprod)"
Write-Host "Classification: SEV-1 - Active data exfiltration and privilege escalation"

# Create incident ticket in Jira
Write-Host "Jira ticket: IAM-SEC-2026-042"
```

---

## 2. Containment (14:33–14:45 UTC)

### Step 1: Disable Compromised Access Keys

```powershell
# scripts/containment/disable-keys.ps1
# Target: svc-cicd-deploy's access keys in Nonproduction account

$profile = "inno-nonprod-admin"
$compromisedUser = "svc-cicd-deploy"

# List all access keys for the compromised user
$keys = aws iam list-access-keys --user-name $compromisedUser --profile $profile `
  | ConvertFrom-Json

foreach ($key in $keys.AccessKeyMetadata) {
  if ($key.Status -eq "Active") {
    aws iam update-access-key --user-name $compromisedUser `
      --access-key-id $key.AccessKeyId --status Inactive --profile $profile
    Write-Host "DEACTIVATED: $($key.AccessKeyId) for $compromisedUser"
  }
}

# Verify no active keys remain
$remaining = aws iam list-access-keys --user-name $compromisedUser --profile $profile `
  | ConvertFrom-Json
Write-Host "Remaining active keys: $(($remaining.AccessKeyMetadata | Where-Object Status -eq 'Active').Count)"
```

### Step 2: Disable Second Compromised User

```powershell
# The attacker created keys for svc-backup-agent. Disable those too.
$secondUser = "svc-backup-agent"

$keys2 = aws iam list-access-keys --user-name $secondUser --profile $profile `
  | ConvertFrom-Json
foreach ($key in $keys2.AccessKeyMetadata) {
  if ($key.Status -eq "Active") {
    aws iam update-access-key --user-name $secondUser `
      --access-key-id $key.AccessKeyId --status Inactive --profile $profile
    Write-Host "DEACTIVATED: $($key.AccessKeyId) for $secondUser"
  }
}

# Also disable any keys created in the last 24 hours for any user
$newKeys = aws iam list-access-keys --user-name $compromisedUser --profile $profile `
  | ConvertFrom-Json
foreach ($key in $newKeys.AccessKeyMetadata) {
  $created = [DateTime]$key.CreateDate
  if (($created -gt (Get-Date).AddHours(-24)) -and ($key.Status -eq "Active")) {
    aws iam update-access-key --user-name $compromisedUser `
      --access-key-id $key.AccessKeyId --status Inactive --profile $profile
    Write-Host "DEACTIVATED (newly created): $($key.AccessKeyId)"
  }
}
```

### Step 3: Terminate Rogue EC2 Instance

```powershell
# Terminate the cryptomining instance in eu-central-1
# Note: The SCP blocked the launch in non-approved regions, but the instance
# exists. Need to assume a role in the Security account that has cross-region access.

$rogueInstanceId = "i-0abcd1234efgh5678"
$rogueRegion = "eu-central-1"

# First, snapshot the volume for forensics
$instanceInfo = aws ec2 describe-instances --instance-ids $rogueInstanceId `
  --region $rogueRegion --profile $profile | ConvertFrom-Json
$volumeId = $instanceInfo.Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId

aws ec2 create-snapshot --volume-id $volumeId --description "Forensic snapshot - IAM-SEC-2026-042" `
  --region $rogueRegion --profile $profile
Write-Host "FORENSIC SNAPSHOT: $volumeId snapshotted"

# Terminate the instance
aws ec2 terminate-instances --instance-ids $rogueInstanceId `
  --region $rogueRegion --profile $profile
Write-Host "TERMINATED: $rogueInstanceId in $rogueRegion"
```

### Step 4: Apply Account-Level SCP (Emergency)

```powershell
# As a safety measure, attach an emergency deny SCP to the Nonproduction account
# to prevent any further actions while investigation continues
# (This is done via AWS Organizations in the Management account)

$scpPolicyId = "p-emergency-contain"
$targetId = "123456789012"  # inno-nonprod account ID
$orgProfile = "inno-mgmt-admin"

# The SCP already exists as a pre-defined policy:
aws organizations attach-policy --policy-id $scpPolicyId --target-id $targetId `
  --profile $orgProfile
Write-Host "EMERGENCY SCP attached to inno-nonprod"
```

Emergency SCP content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EmergencyContain",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/break-glass-admin",
            "arn:aws:iam::*:role/security-automation"
          ]
        }
      }
    }
  ]
}
```

### Step 5: Rotate All Exposed Secrets

```powershell
# Rotate the secrets for both compromised service accounts
aws secretsmanager rotate-secret --secret-id iam/svc-cicd-deploy `
  --rotation-lambda-arn arn:aws:lambda:us-east-1:111122223333:function:rotate-iam-keys

aws secretsmanager rotate-secret --secret-id iam/svc-backup-agent `
  --rotation-lambda-arn arn:aws:lambda:us-east-1:111122223333:function:rotate-iam-keys

Write-Host "SECRETS ROTATED: svc-cicd-deploy, svc-backup-agent"
```

---

## 3. Investigation (14:45–16:00 UTC)

### Step 1: Identify Scope — CloudTrail Analysis

```powershell
# scripts/investigation/analyze-cloudtrail.ps1
$startTime = "2026-07-15T14:00:00Z"
$endTime = "2026-07-15T15:00:00Z"
$compromisedUser = "svc-cicd-deploy"
$targetAccount = "123456789012"

Write-Host "=== CLOUDTRAIL ANALYSIS ==="
Write-Host "Period: $startTime to $endTime"

# Query all actions by the compromised user
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=$compromisedUser `
  --start-time $startTime --end-time $endTime --max-results 50 | ConvertFrom-Json | `
  Select-Object -ExpandProperty Events | Format-Table EventTime, EventName, SourceIPAddress, Resources

# Check if SCP denied any actions (indicates attacker attempts)
Write-Host "`n=== DENIED ACTIONS ==="
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser `
  --start-time $startTime --end-time $endTime --max-results 10 | ConvertFrom-Json | `
  Select-Object -ExpandProperty Events | Format-Table EventTime, EventName, ErrorCode, ErrorMessage
```

### Step 2: GuardDuty Investigation

```powershell
# Get full details of the GuardDuty finding
$findingId = "arn:aws:guardduty:us-east-1:444455556666:detector/abc123/finding/xyz789"

aws guardduty get-findings --detector-id abc123 `
  --finding-ids $findingId --profile inno-security-read | ConvertFrom-Json | `
  Select-Object -ExpandProperty Findings | Format-List

# Check for related findings
aws guardduty list-findings --detector-id abc123 `
  --finding-criteria '{"Criterion": {"severity": {"Gte": 5}}}' `
  --profile inno-security-read
```

### Step 3: S3 Data Exfiltration Analysis

```powershell
# Check which objects were accessed from the compromised keys
$bucketName = "inno-prod-backup-data"

aws s3api get-bucket-location --bucket $bucketName --profile $profile

# List recent object accesses via CloudTrail data events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=$bucketName `
  --start-time $startTime --end-time $endTime | ConvertFrom-Json | `
  Select-Object -ExpandProperty Events | `
  Where-Object { $_.EventName -eq "GetObject" } | `
  Format-Table EventTime, Username, Resources
```

### Step 4: Identify Root Cause

```powershell
Write-Host "=== ROOT CAUSE ANALYSIS ==="
Write-Host ""
Write-Host "Root Cause: The CI/CD pipeline (Jenkins) stored svc-cicd-deploy access keys "
Write-Host "as plaintext environment variables in build configuration. An attacker gained "
Write-Host "access to the Jenkins GUI via a compromised admin account (no MFA on Jenkins) "
Write-Host "and extracted the AWS keys from build logs."
Write-Host ""
Write-Host "Attack Chain:"
Write-Host "1. Attacker accessed Jenkins (no MFA) → viewed build config → extracted svc-cicd-deploy keys"
Write-Host "2. Keys used from Tor exit node to launch GPU instance in unauthorized region"
Write-Host "3. Privilege escalation: created new keys for svc-backup-agent"
Write-Host "4. Data exfiltration: accessed s3://inno-prod-backup-data (~500MB)"
Write-Host "5. Attempted CreateUser — blocked by SCP"
Write-Host ""
Write-Host "Scope:"
Write-Host "- Users compromised: svc-cicd-deploy, svc-backup-agent"
Write-Host "- Data exfiltrated: ~500MB from inno-prod-backup-data"
Write-Host "- Resources created: 1x p3.16xlarge in eu-central-1 (terminated)"
Write-Host "- IAM Identity Center: NOT affected (incident limited to IAM users)"
```

---

## 4. Recovery (16:00–17:00 UTC)

### Step 1: Remove Emergency SCP

```powershell
aws organizations detach-policy --policy-id p-emergency-contain `
  --target-id 123456789012 --profile inno-mgmt-admin
Write-Host "Emergency SCP removed from inno-nonprod"
```

### Step 2: Create New Access Keys for Service Accounts

```powershell
# The rotation Lambda already created new keys.
# Verify they are active:
$newKeys = aws iam list-access-keys --user-name svc-cicd-deploy `
  --profile inno-nonprod-admin | ConvertFrom-Json
foreach ($key in $newKeys.AccessKeyMetadata) {
  Write-Host "svc-cicd-deploy key: $($key.AccessKeyId) - Status: $($key.Status)"
}

# Update CI/CD pipeline with new keys
Write-Host "ACTION REQUIRED: Update Jenkins credentials with new svc-cicd-deploy access keys"
Write-Host "ACTION REQUIRED: Update backup system with new svc-backup-agent access keys"
```

### Step 3: Enable MFA on Jenkins

```powershell
Write-Host "Jenkins MFA enabled via SAML SSO with Entra ID (conditional access: Require MFA)"
Write-Host "Jenkins admin account credentials rotated"
```

---

## 5. Post-Incident Improvements

### Control Improvement 1: SCP — Block Crypto Instance Types

```hcl
# Add to existing SCPs:
{
  "Sid": "BlockCryptoInstances",
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringLike": {
      "ec2:InstanceType": ["p*", "g*", "inf*", "f*"]
    }
  }
}
```

### Control Improvement 2: IAM Access Analyzer — Service Key Rotation Alerts

```hcl
# Enable IAM Access Analyzer to alert on keys older than 90 days
resource "aws_iam_access_analyzer" "this" {
  analyzer_name = "iam-key-analyzer"
  type          = "ACCOUNT"
}

# Config rule for key age
resource "aws_config_config_rule" "iam_key_rotation" {
  name = "iam-access-key-rotated"
  source {
    owner             = "AWS"
    source_identifier = "ACCESS_KEYS_ROTATED"
  }
  input_parameters = jsonencode({
    maxAccessKeyAge = 90
  })
}
```

### Control Improvement 3: GuardDuty — Custom Threat List for Tor Exit Nodes

```powershell
# Download known Tor exit node IPs and upload as a GuardDuty threat list
Invoke-WebRequest -Uri "https://check.torproject.org/exit-addresses" -OutFile "tor-exit-nodes.txt"

# Parse and upload to GuardDuty
$detectorId = aws guardduty list-detectors --profile inno-security-read `
  | ConvertFrom-Json | Select-Object -ExpandProperty DetectorIds

aws guardduty create-threat-intel-set --detector-id $detectorId[0] `
  --name "Tor-Exit-Nodes" --format TXT --location "s3://inno-security-guardduty/threat-lists/tor-exit-nodes.txt" `
  --activate --profile inno-security-admin

Write-Host "Tor exit node list uploaded to GuardDuty"
```

### Control Improvement 4: Remediation Lambda — Auto-Contain

```python
# lambdas/auto-contain-compromised-keys/lambda_function.py
"""Automatically deactivate IAM keys when GuardDuty finds credential exfiltration."""

import boto3
import json
import os

iam = boto3.client("iam")
sns = boto3.client("sns")

SNS_TOPIC = os.environ["SNS_TOPIC_ARN"]

def lambda_handler(event, context):
    """Triggered by EventBridge rule matching HIGH severity credential exfiltration."""
    detail = event.get("detail", {})
    finding_type = detail.get("type", "")

    if "InstanceCredentialExfiltration" not in finding_type:
        return {"statusCode": 200, "body": "Not a credential exfiltration finding" }

    # Extract the compromised user from the finding
    resource_list = detail.get("resource", {}).get("accessKeyDetails", {})
    username = resource_list.get("userName", "")

    if not username:
        return {"statusCode": 400, "body": "No username found in finding"}

    # Deactivate all active keys
    keys = iam.list_access_keys(UserName=username)
    for key in keys.get("AccessKeyMetadata", []):
        if key["Status"] == "Active":
            iam.update_access_key(
                UserName=username,
                AccessKeyId=key["AccessKeyId"],
                Status="Inactive"
            )

    # Notify SOC
    sns.publish(
        TopicArn=SNS_TOPIC,
        Subject=f"Auto-Contain: Keys deactivated for {username}",
        Message=json.dumps({
            "action": "KEYS_DEACTIVATED",
            "username": username,
            "finding_type": finding_type
        })
    )

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": f"Keys deactivated for {username}",
            "keys_deactivated": sum(
                1 for k in keys.get("AccessKeyMetadata", []) if k["Status"] == "Active"
            )
        })
    }
```

### Control Improvement 5: EventBridge Rule for Auto-Response

```hcl
# terraform/incident-response/eventbridge.tf

resource "aws_cloudwatch_event_rule" "credential_exfiltration" {
  name        = "credential-exfiltration-auto-response"
  description = "Trigger auto-containment on credential exfiltration findings"

  event_pattern = jsonencode({
    source = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      type = ["UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration*"]
      severity = [{ numeric = [">=", 7] }]
    }
  })
}

resource "aws_cloudwatch_event_target" "auto_contain" {
  rule      = aws_cloudwatch_event_rule.credential_exfiltration.name
  target_id = "AutoContainLambda"
  arn       = aws_lambda_function.auto_contain.arn
}
```

### Control Improvement 6: Block Regions at SCP Level (Enforce Existing SCP)

```json
{
  "Sid": "EnforceAllowedRegions",
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
    }
  }
}
```

---

## 6. Post-Incident Report Summary

```powershell
Write-Host "=== POST-INCIDENT REPORT: IAM-SEC-2026-042 ==="
Write-Host ""
Write-Host "Incident Summary: Compromised CI/CD service account led to privilege escalation,"
Write-Host "unauthorized EC2 launch (cryptomining), and S3 data exfiltration."
Write-Host ""
Write-Host "Timeline:"
Write-Host "  14:28  - Attacker launches GPU instance in eu-central-1"
Write-Host "  14:29  - Attacker escalates: creates keys for svc-backup-agent"
Write-Host "  14:30  - Attacker exfiltrates data from inno-prod-backup-data"
Write-Host "  14:31  - Attacker attempts CreateUser (DENIED by SCP)"
Write-Host "  14:32  - GuardDuty finding generated"
Write-Host "  14:33  - SOC acknowledges incident (1 min)"
Write-Host "  14:34  - Keys deactivated for svc-cicd-deploy"
Write-Host "  14:35  - Keys deactivated for svc-backup-agent"
Write-Host "  14:36  - Rogue instance terminated (forensic snapshot taken)"
Write-Host "  14:37  - Emergency SCP attached"
Write-Host "  14:40  - Secrets rotated"
Write-Host "  14:45  - CONTAINMENT COMPLETE (13 min)"
Write-Host ""
Write-Host "Root Cause: Jenkins CI/CD server had no MFA; plaintext AWS keys in build config"
Write-Host ""
Write-Host "Impact:"
Write-Host "  - Data exfiltrated: ~500MB backup data (customer PII — notification required)"
Write-Host "  - Cost: ~$2.40 for 8 minutes of p3.16xlarge runtime (before termination)"
Write-Host "  - Downtime: None (service accounts are non-interactive)"
Write-Host ""
Write-Host "Improvements Implemented:"
Write-Host "  1. SCP: Block crypto instance types (p*, g*, inf*, f*)"
Write-Host "  2. IAM Access Analyzer: Alert on keys > 90 days"
Write-Host "  3. GuardDuty: Custom threat list for Tor exit nodes"
Write-Host "  4. Lambda: Auto-contain credential exfiltration findings"
Write-Host "  5. EventBridge: Automated response rule"
Write-Host "  6. Jenkins: MFA enforced via Entra ID SAML SSO"
Write-Host ""
Write-Host "Lessons Learned:"
Write-Host "  - Service account keys must never be stored in CI/CD plaintext"
Write-Host "  - Use IAM Roles Anywhere or OIDC for CI/CD instead of long-lived keys"
Write-Host "  - SCP region restriction works but needs instance-type blocking too"
Write-Host "  - Auto-response Lambda would have saved ~5 minutes of containment time"
Write-Host ""
Write-Host "=== END OF REPORT ==="
```

---

## 7. Console Workflow (Manual Alternative)

### Deactivate Access Keys

1. **AWS Console** → IAM → Users → `svc-cicd-deploy` → **Security credentials**
2. Under **Access keys**, click **Make inactive** for each active key
3. Repeat for `svc-backup-agent`

### Terminate EC2 Instance

1. **AWS Console** → Switch to the region `eu-central-1`
2. **EC2** → Instances → Select `i-0abcd1234efgh5678`
3. **Instance state** → **Terminate**

### Attach Emergency SCP

1. **AWS Console** → AWS Organizations → Policies → **SCP**
2. Click **EmergencyContain**
3. **Attach** → Select Nonproduction account

---

## Summary

| Phase | Action | Time | Duration |
|---|---|---|---|
| Detection | GuardDuty finding received | 14:32 | — |
| Classification | SEV-1 declared, Jira opened | 14:33 | 1 min |
| Containment | Keys deactivated, instance terminated, SCP attached | 14:34–14:45 | 12 min |
| Investigation | CloudTrail analysis, scope identification | 14:45–16:00 | 75 min |
| Recovery | Emergency SCP removed, new keys created | 16:00–17:00 | 60 min |
| Post-Incident | 6 control improvements implemented | 17:00–18:30 | 90 min |

**Total time-to-contain: 13 minutes** (under 15-min SLA)
