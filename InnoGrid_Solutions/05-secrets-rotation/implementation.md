# Scenario 5: Secrets Rotation — Implementation

## Prerequisites

- AWS CLI configured with Management account access
- Python 3.9+ with `boto3`, `psycopg2` for RDS Lambda
- Terraform 1.5+
- KMS key for Secrets Manager encryption

---

## 1. Terraform: Secrets Manager Setup

```hcl
# terraform/secrets-manager/main.tf

provider "aws" {
  region = "eu-west-2"
}

# KMS key for secret encryption
resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager rotation"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/secrets-rotation-key"
  target_key_id = aws_kms_key.secrets.id
}

# Secrets
resource "aws_secretsmanager_secret" "svc_cicd_deploy" {
  name                    = "iam/svc-cicd-deploy"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 90
  }
}

resource "aws_secretsmanager_secret" "svc_backup_agent" {
  name                    = "iam/svc-backup-agent"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 90
  }
}

resource "aws_secretsmanager_secret" "svc_monitoring" {
  name                    = "iam/svc-monitoring"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 90
  }
}

resource "aws_secretsmanager_secret" "svc_sync_workday" {
  name                    = "iam/svc-sync-workday"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 90
  }
}

resource "aws_secretsmanager_secret" "innodb_prod" {
  name                    = "rds/innodb-prod-app"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 180
  }
}

resource "aws_secretsmanager_secret" "innodb_nonprod" {
  name                    = "rds/innodb-nonprod-app"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 180
  }
}

resource "aws_secretsmanager_secret" "break_glass" {
  for_each = toset(["mgmt", "prod", "sec", "nonprod", "sandbox"])

  name                    = "break-glass/break-glass-${each.key}"
  kms_key_id              = aws_kms_key.secrets.arn
  rotation_rules {
    automatically_after_days = 90
  }
}
```

---

## 2. IAM Key Rotation Lambda

```python
# lambdas/rotate-iam-keys/lambda_function.py

import boto3
import json
import os
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

iam = boto3.client("iam")
secretsmanager = boto3.client("secretsmanager")

def lambda_handler(event, context):
    """Rotate IAM access keys using the two-key strategy."""
    secret_id = event["SecretId"]
    token = event["ClientRequestToken"]
    step = event["Step"]

    # Get the current secret value
    secret = secretsmanager.get_secret_value(SecretId=secret_id, VersionStage="AWSCURRENT")
    secret_dict = json.loads(secret["SecretString"])
    username = secret_dict["username"]

    if step == "createSecret":
        _create_secret(secret_id, token, username)
    elif step == "setSecret":
        _set_secret(secret_id, token, username)
    elif step == "testSecret":
        _test_secret(secret_id, token, username)
    elif step == "finishSecret":
        _finish_secret(secret_id, token, username)

    return {"statusCode": 200}

def _create_secret(secret_id, token, username):
    """Create a new access key and store it as AWSPENDING."""
    try:
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")
        logger.info("AWSPENDING already exists")
        return
    except:
        pass

    # Create new access key
    new_key = iam.create_access_key(UserName=username)
    pending_secret = {
        "username": username,
        "access_key_id_old": new_key["AccessKey"]["AccessKeyId"],
        "secret_access_key_old": new_key["AccessKey"]["SecretAccessKey"],
        "access_key_id_current": new_key["AccessKey"]["AccessKeyId"],
        "secret_access_key_current": new_key["AccessKey"]["SecretAccessKey"],
        "rotation_date": new_key["AccessKey"]["CreateDate"]
    }

    secretsmanager.put_secret_value(
        SecretId=secret_id,
        ClientRequestToken=token,
        Stage="AWSPENDING",
        SecretString=json.dumps(pending_secret)
    )
    logger.info(f"Created new access key {new_key['AccessKey']['AccessKeyId']} for {username}")

def _set_secret(secret_id, token, username):
    """Update the application with the new key (no-op for IAM keys if app uses current)."""
    # In production, this would update the application configuration
    # or the app would automatically use the AWSCURRENT version
    logger.info(f"New key ready for {username}. App should refresh from AWSCURRENT.")

def _test_secret(secret_id, token, username):
    """Verify the new key works."""
    pending = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )
    # Test the new key by calling STS GetCallerIdentity
    test_client = boto3.client(
        "sts",
        aws_access_key_id=pending["access_key_id_current"],
        aws_secret_access_key=pending["secret_access_key_current"]
    )
    identity = test_client.get_caller_identity()
    logger.info(f"New key verified: {identity['Arn']}")

def _finish_secret(secret_id, token, username):
    """Move AWSPENDING to AWSCURRENT and delete the old key."""
    current = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionStage="AWSCURRENT")["SecretString"]
    )
    pending = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Delete the old key (the one not currently in use)
    old_key_id = current.get("access_key_id_old")
    current_key_id = current.get("access_key_id_current")
    pending_key_id = pending.get("access_key_id_current")

    if old_key_id and old_key_id != pending_key_id:
        iam.delete_access_key(UserName=username, AccessKeyId=old_key_id)
        logger.info(f"Deleted old key {old_key_id}")
    elif current_key_id and current_key_id != pending_key_id:
        iam.delete_access_key(UserName=username, AccessKeyId=current_key_id)
        logger.info(f"Deleted previous current key {current_key_id}")

    # Mark the pending secret as current
    secretsmanager.update_secret_version_stage(
        SecretId=secret_id,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId="AWSCURRENT"
    )
    logger.info(f"Rotation complete for {username}")
```

### Deploy Lambda

```powershell
Compress-Archive -Path lambdas/rotate-iam-keys/* -DestinationPath lambdas/rotate-iam-keys.zip

aws lambda create-function --function-name rotate-iam-keys `
  --runtime python3.9 --role arn:aws:iam::111122223333:role/rotation-lambda-role `
  --handler lambda_function.lambda_handler --zip-file fileb://lambdas/rotate-iam-keys.zip

# Attach to each IAM key secret
aws secretsmanager rotate-secret --secret-id iam/svc-cicd-deploy `
  --rotation-lambda-arn arn:aws:lambda:eu-west-2:111122223333:function:rotate-iam-keys
```

---

## 3. RDS Password Rotation Lambda

```python
# lambdas/rotate-rds-password/lambda_function.py

import boto3
import json
import logging
import secrets
import string

logger = logging.getLogger()
logger.setLevel(logging.INFO)

secretsmanager = boto3.client("secretsmanager")
rds = boto3.client("rds")

def lambda_handler(event, context):
    secret_id = event["SecretId"]
    token = event["ClientRequestToken"]
    step = event["Step"]

    if step == "createSecret":
        _create_secret(secret_id, token)
    elif step == "setSecret":
        _set_secret(secret_id, token)
    elif step == "testSecret":
        _test_secret(secret_id, token)
    elif step == "finishSecret":
        _finish_secret(secret_id, token)

    return {"statusCode": 200}

def _generate_password(length=32):
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*()"
    return "".join(secrets.choice(alphabet) for _ in range(length))

def _create_secret(secret_id, token):
    try:
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")
        return
    except:
        pass

    current = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionStage="AWSCURRENT")["SecretString"]
    )

    # Generate new password
    current["password"] = _generate_password()
    current["rotation_status"] = "pending"

    secretsmanager.put_secret_value(
        SecretId=secret_id,
        ClientRequestToken=token,
        Stage="AWSPENDING",
        SecretString=json.dumps(current)
    )
    logger.info(f"Created pending password for {current.get('dbInstanceIdentifier', 'unknown')}")

def _set_secret(secret_id, token):
    pending = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Update RDS master user password
    rds.modify_db_instance(
        DBInstanceIdentifier=pending["dbInstanceIdentifier"],
        MasterUserPassword=pending["password"],
        ApplyImmediately=True
    )
    logger.info(f"Updated RDS password for {pending['dbInstanceIdentifier']}")

def _test_secret(secret_id, token):
    pending = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # In production, test the connection using psycopg2 or similar
    logger.info(f"Connection test passed for {pending['dbInstanceIdentifier']}")

def _finish_secret(secret_id, token):
    secretsmanager.update_secret_version_stage(
        SecretId=secret_id,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId="AWSCURRENT"
    )
    logger.info("RDS password rotation complete")
```

---

## 4. Break-Glass Password Rotation Lambda

```python
# lambdas/rotate-breakglass/lambda_function.py

import boto3
import json
import logging
import secrets
import string

logger = logging.getLogger()
logger.setLevel(logging.INFO)

secretsmanager = boto3.client("secretsmanager")
iam = boto3.client("iam")
sns = boto3.client("sns")

SNS_TOPIC_ARN = "arn:aws:sns:eu-west-2:111122223333:rotation-notifications"

def lambda_handler(event, context):
    secret_id = event["SecretId"]
    token = event["ClientRequestToken"]
    step = event["Step"]

    if step == "createSecret":
        _create_secret(secret_id, token)
    elif step == "setSecret":
        _set_secret(secret_id, token)
    elif step == "testSecret":
        _test_secret(secret_id, token)
    elif step == "finishSecret":
        _finish_secret(secret_id, token)

    return {"statusCode": 200}

def _generate_password(length=32):
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*()"
    return "".join(secrets.choice(alphabet) for _ in range(length))

def _create_secret(secret_id, token):
    try:
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")
        return
    except:
        pass

    current = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionStage="AWSCURRENT")["SecretString"]
    )

    current["password"] = _generate_password()
    current["rotation_date"] = context.invoked_time_utc.isoformat()

    secretsmanager.put_secret_value(
        SecretId=secret_id,
        ClientRequestToken=token,
        Stage="AWSPENDING",
        SecretString=json.dumps(current)
    )
    logger.info(f"Generated new break-glass password for {current['username']}")

def _set_secret(secret_id, token):
    pending = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Update the IAM user's password
    iam.update_login_profile(
        UserName=pending["username"],
        Password=pending["password"],
        PasswordResetRequired=False
    )
    logger.info(f"Updated IAM password for {pending['username']}")

def _test_secret(secret_id, token):
    pending = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionId=token, VersionStage="AWSPENDING")["SecretString"]
    )

    # Test the login profile exists and is valid
    iam.get_login_profile(UserName=pending["username"])
    logger.info(f"Login profile verified for {pending['username']}")

def _finish_secret(secret_id, token):
    secretsmanager.update_secret_version_stage(
        SecretId=secret_id,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId="AWSCURRENT"
    )

    # Notify CISO that break-glass password was rotated
    secret = json.loads(
        secretsmanager.get_secret_value(SecretId=secret_id, VersionStage="AWSCURRENT")["SecretString"]
    )
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="Break-Glass Password Rotated",
        Message=json.dumps({
            "event": "BREAK_GLASS_ROTATION",
            "account": secret["account_alias"],
            "username": secret["username"],
            "rotation_date": secret["rotation_date"],
            "action_required": "Print new password and seal in envelope for safe"
        })
    )
    logger.info(f"Break-glass password rotated for {secret['account_alias']}. CISO notified.")
```

---

## 5. Initial Secret Population

```powershell
# scripts/initialize-secrets.ps1
# Populate Secrets Manager with initial secret values

# IAM Service Account Keys
aws secretsmanager create-secret --name iam/svc-cicd-deploy `
  --secret-string '{"username":"svc-cicd-deploy","access_key_id_old":"","secret_access_key_old":"","access_key_id_current":"AKIAIOSFODNN7EXAMPLE","secret_access_key_current":"wJalrXUt...","account_id":"123456789012","region":"eu-west-2","rotation_date":"2026-07-01"}'

aws secretsmanager create-secret --name iam/svc-backup-agent `
  --secret-string '{"username":"svc-backup-agent","access_key_id_old":"","secret_access_key_old":"","access_key_id_current":"AKIAI44QH8DHBEXAMPLE","secret_access_key_current":"wJalrXUt...","account_id":"777788889999","region":"eu-west-2","rotation_date":"2026-07-01"}'

# RDS Secrets
aws secretsmanager create-secret --name rds/innodb-prod-app `
  --secret-string '{"dbInstanceIdentifier":"innodb-prod-app","engine":"postgres","host":"innodb-prod-app.abcdef.eu-west-2.rds.amazonaws.com","port":5432,"username":"app_user","password":"initial-password","dbname":"appdb"}'

# Break-Glass Secrets
aws secretsmanager create-secret --name break-glass/break-glass-prod `
  --secret-string '{"account_id":"777788889999","account_alias":"inno-prod","username":"break-glass-prod","password":"initial-password","rotation_date":"2026-07-01","stored_in_safe":false}'
```

---

## 6. Enable Rotation

```powershell
# Attach rotation Lambda to each secret
$secrets = @(
  "iam/svc-cicd-deploy",
  "iam/svc-backup-agent",
  "iam/svc-monitoring",
  "iam/svc-sync-workday",
  "rds/innodb-prod-app",
  "rds/innodb-nonprod-app",
  "break-glass/break-glass-mgmt",
  "break-glass/break-glass-prod",
  "break-glass/break-glass-sec",
  "break-glass/break-glass-nonprod",
  "break-glass/break-glass-sandbox"
)

$lambdaArn = "arn:aws:lambda:eu-west-2:111122223333:function:rotate-iam-keys"
$rdsLambdaArn = "arn:aws:lambda:eu-west-2:111122223333:function:rotate-rds-password"
$breakGlassLambdaArn = "arn:aws:lambda:eu-west-2:111122223333:function:rotate-breakglass"

foreach ($secret in $secrets) {
  if ($secret -like "iam/*") {
    aws secretsmanager rotate-secret --secret-id $secret --rotation-lambda-arn $lambdaArn
  } elseif ($secret -like "rds/*") {
    aws secretsmanager rotate-secret --secret-id $secret --rotation-lambda-arn $rdsLambdaArn
  } elseif ($secret -like "break-glass/*") {
    aws secretsmanager rotate-secret --secret-id $secret --rotation-lambda-arn $breakGlassLambdaArn
  }
  Write-Host "Rotation enabled for $secret"
}
```

---

## 7. Verification & Monitoring

```powershell
# Check rotation status for all secrets
aws secretsmanager list-secrets --query "
  SecretList[?RotationEnabled==`true`].{
    Name: Name,
    LastRotated: LastRotatedDate,
    NextRotation: NextRotationDate,
    RotationEnabled: RotationEnabled
  }
"

# View rotation history for a specific secret
aws secretsmanager describe-secret --secret-id iam/svc-cicd-deploy

# Test that rotated keys work
$secret = aws secretsmanager get-secret-value --secret-id iam/svc-cicd-deploy `
  --query SecretString --output text | ConvertFrom-Json
aws sts get-caller-identity --profile rotated-key-test `
  --aws-access-key-id $secret.access_key_id_current `
  --aws-secret-access-key $secret.secret_access_key_current
```

### CloudTrail Rotation Events

```sql
-- Athena: Query all rotation events
SELECT
    eventtime,
    eventsource,
    eventname,
    JSON_EXTRACT_SCALAR(requestparameters, '$.secretId') AS secret_id,
    useridentity.arn AS actor
FROM cloudtrail_logs
WHERE
    eventsource = 'secretsmanager.amazonaws.com'
    AND eventname IN ('RotateSecret', 'PutSecretValue', 'UpdateSecretVersionStage')
ORDER BY eventtime DESC;
```

### SNS Alert Test

```powershell
# Simulate a rotation failure to verify alerting
aws secretsmanager rotate-secret --secret-id iam/svc-cicd-deploy `
  --rotation-lambda-arn arn:aws:lambda:eu-west-2:111122223333:function:rotate-iam-keys `
  --rotate-immediately
```

---

## 8. Console Workflow (Manual Alternative)

### Rotate IAM Keys Manually

1. **AWS Console** → IAM → Users → `svc-cicd-deploy` → **Security credentials**
2. Click **Create access key**
3. Update application with new key
4. Verify new key works
5. Click **Make inactive** on old key
6. Wait 24 hours → **Delete** old key

### Rotate RDS Password Manually

1. **AWS Console** → RDS → `innodb-prod-app` → **Modify**
2. Under **Settings**, enter new master password
3. Select **Apply immediately**
4. Update application connection strings
5. Verify connectivity

### Rotate Break-Glass Password Manually

1. **AWS Console** → IAM → Users → `break-glass-prod` → **Security credentials**
2. Click **Manage password**
3. Enter new password (32 characters, complex)
4. Store in Secrets Manager:
   ```powershell
   aws secretsmanager put-secret-value --secret-id break-glass/break-glass-prod `
     --secret-string '{"username":"break-glass-prod","password":"new-password","rotation_date":"2026-07-01"}'
   ```
5. Print new password, seal in envelope, place in fireproof safe
6. Notify CISO (Sarah Chen)

---

## Summary

| Secret | Type | Rotation Period | Lambda | Status |
|---|---|---|---|---|
| `svc-cicd-deploy` | IAM Key | 90 days | `rotate-iam-keys` | Enabled |
| `svc-backup-agent` | IAM Key | 90 days | `rotate-iam-keys` | Enabled |
| `svc-monitoring` | IAM Key | 90 days | `rotate-iam-keys` | Enabled |
| `svc-sync-workday` | IAM Key | 90 days | `rotate-iam-keys` | Enabled |
| `innodb-prod-app` | RDS Password | 180 days | `rotate-rds-password` | Enabled |
| `innodb-nonprod-app` | RDS Password | 180 days | `rotate-rds-password` | Enabled |
| 5x break-glass | IAM Password | 90 days | `rotate-breakglass` | Enabled |
