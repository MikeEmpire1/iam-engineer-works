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

```bash
# Timestamp: 2026-07-15 14:33 UTC
echo "=== INCIDENT ACKNOWLEDGED ==="
echo "Finding: UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS"
echo "Severity: HIGH (7.0)"
echo "User: svc-cicd-deploy (inno-nonprod)"
echo "Classification: SEV-1 - Active data exfiltration and privilege escalation"

# Create incident ticket in Jira
echo "Jira ticket: IAM-SEC-2026-042"
```

---

## 2. Containment (14:33–14:45 UTC)

### Step 1: Disable Compromised Access Keys

```bash
# scripts/containment/disable-keys.sh
# Target: svc-cicd-deploy's access keys in Nonproduction account

profile="inno-nonprod-admin"
compromisedUser="svc-cicd-deploy"

# List all access keys for the compromised user
keys=$(aws iam list-access-keys --user-name "$compromisedUser" --profile "$profile")

echo "$keys" | jq -r '.AccessKeyMetadata[] | select(.Status == "Active") | .AccessKeyId' | while read keyId; do
  aws iam update-access-key --user-name "$compromisedUser" \
    --access-key-id "$keyId" --status Inactive --profile "$profile"
  echo "DEACTIVATED: $keyId for $compromisedUser"
done

# Verify no active keys remain
remaining=$(aws iam list-access-keys --user-name "$compromisedUser" --profile "$profile")
activeCount=$(echo "$remaining" | jq '[.AccessKeyMetadata[] | select(.Status == "Active")] | length')
echo "Remaining active keys: $activeCount"
```

### Step 2: Disable Second Compromised User

```bash
# The attacker created keys for svc-backup-agent. Disable those too.
secondUser="svc-backup-agent"

keys2=$(aws iam list-access-keys --user-name "$secondUser" --profile "$profile")
echo "$keys2" | jq -r '.AccessKeyMetadata[] | select(.Status == "Active") | .AccessKeyId' | while read keyId; do
  aws iam update-access-key --user-name "$secondUser" \
    --access-key-id "$keyId" --status Inactive --profile "$profile"
  echo "DEACTIVATED: $keyId for $secondUser"
done

# Also disable any keys created in the last 24 hours for any user
newKeys=$(aws iam list-access-keys --user-name "$compromisedUser" --profile "$profile")
cutoff=$(date -u -d '24 hours ago' '+%Y-%m-%dT%H:%M:%SZ')
echo "$newKeys" | jq -r --arg cutoff "$cutoff" '.AccessKeyMetadata[] | select(.Status == "Active") | select(.CreateDate >= $cutoff) | .AccessKeyId' | while read keyId; do
  aws iam update-access-key --user-name "$compromisedUser" \
    --access-key-id "$keyId" --status Inactive --profile "$profile"
  echo "DEACTIVATED (newly created): $keyId"
done
```

### Step 3: Terminate Rogue EC2 Instance

```bash
# Terminate the cryptomining instance in ap-southeast-1
# Note: The SCP blocked the launch in non-approved regions, but the instance
# exists. Need to assume a role in the Security account that has cross-region access.

rogueInstanceId="i-0abcd1234efgh5678"
rogueRegion="ap-southeast-1"

# First, snapshot the volume for forensics
instanceInfo=$(aws ec2 describe-instances --instance-ids "$rogueInstanceId" \
  --region "$rogueRegion" --profile "$profile")
volumeId=$(echo "$instanceInfo" | jq -r '.Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId')

aws ec2 create-snapshot --volume-id "$volumeId" --description "Forensic snapshot - IAM-SEC-2026-042" \
  --region "$rogueRegion" --profile "$profile"
echo "FORENSIC SNAPSHOT: $volumeId snapshotted"

# Terminate the instance
aws ec2 terminate-instances --instance-ids "$rogueInstanceId" \
  --region "$rogueRegion" --profile "$profile"
echo "TERMINATED: $rogueInstanceId in $rogueRegion"
```

### Step 4: Apply Account-Level SCP (Emergency)

```bash
# As a safety measure, attach an emergency deny SCP to the Nonproduction account
# to prevent any further actions while investigation continues
# (This is done via AWS Organizations in the Management account)

scpPolicyId="p-emergency-contain"
targetId="123456789012"  # inno-nonprod account ID
orgProfile="inno-mgmt-admin"

# The SCP already exists as a pre-defined policy:
aws organizations attach-policy --policy-id "$scpPolicyId" --target-id "$targetId" \
  --profile "$orgProfile"
echo "EMERGENCY SCP attached to inno-nonprod"
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

```bash
# Rotate the secrets for both compromised service accounts
aws secretsmanager rotate-secret --secret-id iam/svc-cicd-deploy \
  --rotation-lambda-arn arn:aws:lambda:eu-west-2:111122223333:function:rotate-iam-keys

aws secretsmanager rotate-secret --secret-id iam/svc-backup-agent \
  --rotation-lambda-arn arn:aws:lambda:eu-west-2:111122223333:function:rotate-iam-keys

echo "SECRETS ROTATED: svc-cicd-deploy, svc-backup-agent"
```

---

## 3. Investigation (14:45–16:00 UTC)

### Step 1: Identify Scope — CloudTrail Analysis

```bash
# scripts/investigation/analyze-cloudtrail.sh
startTime="2026-07-15T14:00:00Z"
endTime="2026-07-15T15:00:00Z"
compromisedUser="svc-cicd-deploy"
targetAccount="123456789012"

echo "=== CLOUDTRAIL ANALYSIS ==="
echo "Period: $startTime to $endTime"

# Query all actions by the compromised user
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue="$compromisedUser" \
  --start-time "$startTime" --end-time "$endTime" --max-results 50 | jq '.Events[] | {EventTime, EventName, SourceIPAddress, Resources}'

# Check if SCP denied any actions (indicates attacker attempts)
echo ""
echo "=== DENIED ACTIONS ==="
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser \
  --start-time "$startTime" --end-time "$endTime" --max-results 10 | jq '.Events[] | {EventTime, EventName, ErrorCode, ErrorMessage}'
```

### Step 2: GuardDuty Investigation

```bash
# Get full details of the GuardDuty finding
findingId="arn:aws:guardduty:eu-west-2:444455556666:detector/abc123/finding/xyz789"

aws guardduty get-findings --detector-id abc123 \
  --finding-ids "$findingId" --profile inno-security-read | jq '.Findings[]'

# Check for related findings
aws guardduty list-findings --detector-id abc123 \
  --finding-criteria '{"Criterion": {"severity": {"Gte": 5}}}' \
  --profile inno-security-read
```

### Step 3: S3 Data Exfiltration Analysis

```bash
# Check which objects were accessed from the compromised keys
bucketName="inno-prod-backup-data"

aws s3api get-bucket-location --bucket "$bucketName" --profile "$profile"

# List recent object accesses via CloudTrail data events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue="$bucketName" \
  --start-time "$startTime" --end-time "$endTime" | jq '.Events[] | select(.EventName == "GetObject") | {EventTime, Username, Resources}'
```

### Step 4: Identify Root Cause

```bash
echo "=== ROOT CAUSE ANALYSIS ==="
echo ""
echo "Root Cause: The CI/CD pipeline (Jenkins) stored svc-cicd-deploy access keys "
echo "as plaintext environment variables in build configuration. An attacker gained "
echo "access to the Jenkins GUI via a compromised admin account (no MFA on Jenkins) "
echo "and extracted the AWS keys from build logs."
echo ""
echo "Attack Chain:"
echo "1. Attacker accessed Jenkins (no MFA) → viewed build config → extracted svc-cicd-deploy keys"
echo "2. Keys used from Tor exit node to launch GPU instance in unauthorised region"
echo "3. Privilege escalation: created new keys for svc-backup-agent"
echo "4. Data exfiltration: accessed s3://inno-prod-backup-data (~500MB)"
echo "5. Attempted CreateUser — blocked by SCP"
echo ""
echo "Scope:"
echo "- Users compromised: svc-cicd-deploy, svc-backup-agent"
echo "- Data exfiltrated: ~500MB from inno-prod-backup-data"
echo "- Resources created: 1x p3.16xlarge in ap-southeast-1 (terminated)"
echo "- IAM Identity Centre: NOT affected (incident limited to IAM users)"
```

---

## 4. Recovery (16:00–17:00 UTC)

### Step 1: Remove Emergency SCP

```bash
aws organizations detach-policy --policy-id p-emergency-contain \
  --target-id 123456789012 --profile inno-mgmt-admin
echo "Emergency SCP removed from inno-nonprod"
```

### Step 2: Create New Access Keys for Service Accounts

```bash
# The rotation Lambda already created new keys.
# Verify they are active:
newKeys=$(aws iam list-access-keys --user-name svc-cicd-deploy \
  --profile inno-nonprod-admin)
echo "$newKeys" | jq -r '.AccessKeyMetadata[] | "svc-cicd-deploy key: \(.AccessKeyId) - Status: \(.Status)"'

# Update CI/CD pipeline with new keys
echo "ACTION REQUIRED: Update Jenkins credentials with new svc-cicd-deploy access keys"
echo "ACTION REQUIRED: Update backup system with new svc-backup-agent access keys"
```

### Step 3: Enable MFA on Jenkins

```bash
echo "Jenkins MFA enabled via SAML SSO with Entra ID (conditional access: Require MFA)"
echo "Jenkins admin account credentials rotated"
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

### Control Improvement 2: IAM Access Analyser — Service Key Rotation Alerts

```hcl
# Enable IAM Access Analyser to alert on keys older than 90 days
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

```bash
# Download known Tor exit node IPs and upload as a GuardDuty threat list
curl -s -o "tor-exit-nodes.txt" "https://check.torproject.org/exit-addresses"

# Parse and upload to GuardDuty
detectorId=$(aws guardduty list-detectors --profile inno-security-read | jq -r '.DetectorIds[0]')

aws guardduty create-threat-intel-set --detector-id "$detectorId" \
  --name "Tor-Exit-Nodes" --format TXT --location "s3://inno-security-guardduty/threat-lists/tor-exit-nodes.txt" \
  --activate --profile inno-security-admin

echo "Tor exit node list uploaded to GuardDuty"
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
      "aws:RequestedRegion": ["eu-west-2", "eu-west-1"]
    }
  }
}
```

---

## 6. Post-Incident Report Summary

```bash
echo "=== POST-INCIDENT REPORT: IAM-SEC-2026-042 ==="
echo ""
echo "Incident Summary: Compromised CI/CD service account led to privilege escalation,"
echo "unauthorised EC2 launch (cryptomining), and S3 data exfiltration."
echo ""
echo "Timeline:"
echo "  14:28  - Attacker launches GPU instance in ap-southeast-1"
echo "  14:29  - Attacker escalates: creates keys for svc-backup-agent"
echo "  14:30  - Attacker exfiltrates data from inno-prod-backup-data"
echo "  14:31  - Attacker attempts CreateUser (DENIED by SCP)"
echo "  14:32  - GuardDuty finding generated"
echo "  14:33  - SOC acknowledges incident (1 min)"
echo "  14:34  - Keys deactivated for svc-cicd-deploy"
echo "  14:35  - Keys deactivated for svc-backup-agent"
echo "  14:36  - Rogue instance terminated (forensic snapshot taken)"
echo "  14:37  - Emergency SCP attached"
echo "  14:40  - Secrets rotated"
echo "  14:45  - CONTAINMENT COMPLETE (13 min)"
echo ""
echo "Root Cause: Jenkins CI/CD server had no MFA; plaintext AWS keys in build config"
echo ""
echo "Impact:"
echo "  - Data exfiltrated: ~500MB backup data (customer PII — notification required)"
echo "  - Cost: ~$2.40 for 8 minutes of p3.16xlarge runtime (before termination)"
echo "  - Downtime: None (service accounts are non-interactive)"
echo ""
echo "Improvements Implemented:"
echo "  1. SCP: Block crypto instance types (p*, g*, inf*, f*)"
echo "  2. IAM Access Analyser: Alert on keys > 90 days"
echo "  3. GuardDuty: Custom threat list for Tor exit nodes"
echo "  4. Lambda: Auto-contain credential exfiltration findings"
echo "  5. EventBridge: Automated response rule"
echo "  6. Jenkins: MFA enforced via Entra ID SAML SSO"
echo ""
echo "Lessons Learned:"
echo "  - Service account keys must never be stored in CI/CD plaintext"
echo "  - Use IAM Roles Anywhere or OIDC for CI/CD instead of long-lived keys"
echo "  - SCP region restriction works but needs instance-type blocking too"
echo "  - Auto-response Lambda would have saved ~5 minutes of containment time"
echo ""
echo "=== END OF REPORT ==="
```

---

## 7. Console Workflow (Manual Alternative)

### Deactivate Access Keys

1. **AWS Console** → IAM → Users → `svc-cicd-deploy` → **Security credentials**
2. Under **Access keys**, click **Make inactive** for each active key
3. Repeat for `svc-backup-agent`

### Terminate EC2 Instance

1. **AWS Console** → Switch to the region `ap-southeast-1`
2. **EC2** → Instances → Select `i-0abcd1234efgh5678`
3. **Instance state** → **Terminate**

### Attach Emergency SCP

1. **AWS Console** → AWS Organisations → Policies → **SCP**
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
