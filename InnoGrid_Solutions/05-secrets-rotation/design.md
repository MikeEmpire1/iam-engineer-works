# Scenario 5: Secrets Rotation — Design

## Architecture Overview

AWS Secrets Manager handles all secret storage and rotation with Lambda-backed rotation functions:

```
┌──────────────────────────────────────────────────────────────────┐
│                     AWS Secrets Manager                          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ IAM Keys     │  │ RDS          │  │ Break-Glass          │  │
│  │ (90-day)     │  │ Passwords    │  │ Passwords            │  │
│  │              │  │ (180-day)    │  │ (90-day)             │  │
│  │ svc-cicd-    │  │ innodb-prod  │  │ mgmt / prod / sec    │  │
│  │ deploy       │  │ innodb-non   │  │ nonprod / sandbox    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                     │               │
│         ▼                 ▼                     ▼               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Lambda:      │  │ Lambda:      │  │ Lambda:              │  │
│  │ rotate-iam-  │  │ rotate-rds-  │  │ rotate-breakglass    │  │
│  │ keys         │  │ password     │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │  SNS Topic   │
                  │  (failures)  │
                  └──────┬───────┘
                         │
                         ▼
                  ┌──────────────┐
                  │  PagerDuty   │
                  │  (IAM on-    │
                  │  call)       │
                  └──────────────┘
```

## Rotation Strategy

### IAM Access Keys — Two-Key Strategy

```mermaid
flowchart TD
    A[Rotation triggered] --> B[Create new access key]
    B --> C[Update application with new key]
    C --> D[Wait for propagation]
    D --> E[Verify new key works]
    E --> F[Delete old access key]
    F --> G[Update secret in Secrets Manager]
    G --> H[Log to CloudTrail]
```

This ensures zero downtime: the old key remains active until the new key is verified.

### RDS Passwords — Staged Rotation

```mermaid
flowchart TD
    A[Rotation triggered] --> B[Generate new password]
    B --> C[Create alternate user with new password]
    C --> D[Update connection string in app config]
    D --> E[Test connection with new password]
    E --> F[Drop old user / update master password]
    F --> G[Update secret in Secrets Manager]
    G --> H[Log to CloudTrail]
```

### Break-Glass Passwords — Direct Rotation

```mermaid
flowchart TD
    A[Rotation triggered] --> B[Generate new 32-char password]
    B --> C[Update IAM user password]
    C --> D[Store new password in Secrets Manager]
    D --> E[Encrypt with KMS]
    E --> F[SNS notification to CISO: password updated]
    F --> G[Print new password + seal in envelope for safe]
```

## Secret Structure in Secrets Manager

### IAM Key Secret

```json
{
  "username": "svc-cicd-deploy",
  "access_key_id_old": "AKIAIOSFODNN7EXAMPLE",
  "secret_access_key_old": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "access_key_id_current": "AKIAI44QH8DHBEXAMPLE",
  "secret_access_key_current": "wJalrXUtnenFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "account_id": "123456789012",
  "region": "us-east-1",
  "rotation_date": "2026-07-01T00:00:00Z"
}
```

### RDS Secret

```json
{
  "dbInstanceIdentifier": "innodb-prod-app",
  "engine": "postgres",
  "host": "innodb-prod-app.abcdef123456.us-east-1.rds.amazonaws.com",
  "port": 5432,
  "username": "app_user",
  "password": "new-generated-password",
  "dbname": "appdb"
}
```

### Break-Glass Secret

```json
{
  "account_id": "777788889999",
  "account_alias": "inno-prod",
  "username": "break-glass-prod",
  "password": "new-32-char-password",
  "rotation_date": "2026-07-01T00:00:00Z",
  "stored_in_safe": false
}
```

## Rotation Schedule

| Secret | Schedule | Cron Expression | Lambda |
|---|---|---|---|
| IAM service account keys | Every 90 days | `0 0 1 */3 ? *` | `rotate-iam-keys` |
| RDS passwords | Every 180 days | `0 0 1 */6 ? *` | `rotate-rds-password` |
| Break-glass passwords | Every 90 days | `0 0 1 */3 ? *` | `rotate-breakglass` |
| API keys (third-party) | Every 30 days | `0 0 1 * ? *` | `rotate-api-keys` |

## IAM Permissions for Rotation Lambda

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:PutSecretValue",
        "secretsmanager:UpdateSecretVersionStage"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:UpdateAccessKey",
        "iam:DeleteAccessKey",
        "iam:ListAccessKeys"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds:ModifyDBInstance",
        "rds:DescribeDBInstances"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:*:*:rotation-failures"
    }
  ]
}
```

## Compliance Mapping

| Requirement | Control | How It's Met |
|---|---|---|
| SOC 2 CC6.1 | Logical access | Keys rotated on schedule reduces exposure window |
| SOC 2 CC7.2 | Monitoring | Rotation failure alerts via SNS → PagerDuty |
| ISO 27001 A.9.2.4 | Management of secrets | Secrets Manager with automatic rotation |
| ISO 27001 A.10.1.1 | Cryptographic controls | Secrets encrypted with KMS at rest and in transit |
| SOC 2 CC6.7 | Data disposal | Old keys deleted after rotation confirmation |
