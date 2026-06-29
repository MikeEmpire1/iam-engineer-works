# InnoGrid Solutions — AWS Account Structure

## Organizational Units (OUs) Layout

```
AWS Organizations Root (innoGrid-solutions.com)
├── Security OU
│   └── Security (audit, logging, detective controls)
├── Infrastructure OU
│   ├── Production (customer-facing workloads)
│   └── Nonproduction (dev, staging, test)
├── Sandbox OU
│   └── Sandbox (individual engineer experimentation)
└── Management OU (delegated admin)
    └── Management (org root, billing, IAM Identity Center)
```

## Account Details

| Account | AWS Account ID (Alias) | OU | Purpose | Key Services |
|---|---|---|---|---|
| **Management** | `1111-2222-3333` (`inno-mgmt`) | Management | Organization root, billing consolidation, IAM Identity Center, CloudTrail org trail | AWS Organizations, IAM Identity Center, Consolidated Billing, S3 (access logs) |
| **Security** | `4444-5555-6666` (`inno-security`) | Security | Centralized security monitoring and auditing | Amazon GuardDuty, Security Hub, AWS Config, CloudTrail, KMS, Access Analyzer, Detective |
| **Production** | `7777-8888-9999` (`inno-prod`) | Infrastructure | Customer-facing production services | EC2, ECS/EKS, RDS, S3, CloudFront, WAF, Secrets Manager |
| **Nonproduction** | `1234-5678-9012` (`inno-nonprod`) | Infrastructure | Development, staging, and integration test environments | EC2, ECS, RDS (dev), S3, Secrets Manager |
| **Sandbox** | `9876-5432-1098` (`inno-sandbox`) | Sandbox | Individual engineer experimentation, no production data | Limited EC2, limited S3, no RDS |

## Access Model

### Authentication
- **IAM Identity Center** is the single pane of glass for AWS access
- Entra ID users (HR, Finance, Legal, Exec) are synced via SCIM to IAM Identity Center
- Engineering & IT users are managed directly in IAM Identity Center
- Sales & Marketing (guest accounts) have no AWS access by default

### Authorization
- Access is group-based via **Permission Sets** in IAM Identity Center
- Groups follow RBAC patterns (detailed in Scenario 3)
- Production access requires **just-in-time (JIT)** elevation with approval
- No standing IAM users or access keys for humans (except break-glass)

### Cross-Account Architecture
```
                    ┌──────────────────┐
                    │  IAM Identity    │
                    │  Center          │
                    │  (Management)    │
                    └────────┬─────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
    ┌────────────┐   ┌────────────┐   ┌────────────┐
    │ Security   │   │Production  │   │Nonprod     │
    │ (read-only)│   │ (JIT/admin)│   │ (dev/admin)│
    └────────────┘   └────────────┘   └────────────┘
```

### Service Control Policies (SCPs)
InnoGrid enforces two layers of SCP guardrails: **global** (applied to all accounts via the root) and **OU-specific** (service boundaries per OU).

#### Global SCPs (applied to Root)

1. **Root activity block** — No root user actions except under break-glass procedure
2. **Region restriction** — Only `us-east-1`, `us-west-2`, `eu-west-1` allowed (compliance regions)
3. **No unapproved instance types** — Only `t3.*`, `m5.*`, `c5.*`, `r5.*` families allowed
4. **Require encryption** — EBS volumes, S3 buckets, and RDS instances must be encrypted
5. **No public S3 buckets** — `s3:PutBucketPublicAccessBlock` enforced
6. **JIT for production** — No standing `AdministratorAccess` in production account

#### OU-Specific Service Boundary SCPs

These SCPs enforce the **Key Services** column from the account table above, ensuring each OU only runs services consistent with its purpose.

| # | SCP Name | Target OU | Strategy |
|---|----------|-----------|----------|
| 7 | `Deny-NonManagementServices` | Management | Allow-list: deny all services except management/identity/billing/logging |
| 8 | `Deny-NonSecurityServices` | Security | Allow-list: deny all services except security monitoring/auditing |
| 9 | `Deny-SecurityToolTampering` | Infrastructure | Deny-list: block disabling or reconfiguring security tools from workload accounts |
| 10 | `Deny-ProductionResourcesInNonprod` | Nonproduction | Deny-list: block production-grade instance types and RDS classes |
| 11 | `Deny-PersistentResourcesInSandbox` | Sandbox | Deny-list: block RDS, Secrets Manager creation, and persistent storage |

##### Policy 7 — `Deny-NonManagementServices` (Management OU)

Denies all AWS services except those required for organization management, identity, billing, and audit logging. Excludes the break-glass role so emergency access remains functional.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonManagementServices",
      "Effect": "Deny",
      "NotAction": [
        "organizations:*",
        "identitystore:*",
        "sso:*",
        "sso-directory:*",
        "billing:*",
        "aws-portal:*",
        "account:*",
        "s3:*",
        "cloudtrail:*",
        "iam:Get*",
        "iam:List*",
        "iam:CreateServiceLinkedRole",
        "iam:PassRole",
        "kms:Describe*",
        "kms:List*",
        "kms:Get*",
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:CreateKey",
        "kms:ScheduleKeyDeletion",
        "sns:CreateTopic",
        "sns:Subscribe",
        "sns:Get*",
        "sns:List*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/break-glass"
        }
      }
    }
  ]
}
```

##### Policy 8 — `Deny-NonSecurityServices` (Security OU)

Denies all AWS services except those used for security monitoring, detection, and remediation. Prevents compute or data services from running in the security account.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonSecurityServices",
      "Effect": "Deny",
      "NotAction": [
        "guardduty:*",
        "securityhub:*",
        "config:*",
        "cloudtrail:*",
        "kms:*",
        "access-analyzer:*",
        "detective:*",
        "s3:*",
        "sns:*",
        "iam:Get*",
        "iam:List*",
        "iam:PassRole",
        "iam:CreateServiceLinkedRole",
        "events:*",
        "logs:*"
      ],
      "Resource": "*"
    }
  ]
}
```

##### Policy 9 — `Deny-SecurityToolTampering` (Infrastructure OU)

Prevents workload accounts (Production and Nonproduction) from disabling or reconfiguring security tools that are centrally managed from the Security account.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyGuardDutyTampering",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:UpdateDetector",
        "guardduty:DeleteFilter",
        "guardduty:UpdateFilter",
        "guardduty:DisassociateFromAdministratorAccount",
        "guardduty:StopMonitoringMembers",
        "guardduty:UpdateOrganizationConfiguration"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenySecurityHubTampering",
      "Effect": "Deny",
      "Action": [
        "securityhub:DisableSecurityHub",
        "securityhub:UpdateOrganizationConfiguration",
        "securityhub:BatchDisableStandards",
        "securityhub:DeleteActionTarget",
        "securityhub:UpdateActionTarget"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyConfigTampering",
      "Effect": "Deny",
      "Action": [
        "config:DeleteConfigRule",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel",
        "config:StopConfigurationRecorder",
        "config:PutDeliveryChannel",
        "config:PutConfigurationRecorder"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyCloudTrailTampering",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail",
        "cloudtrail:PutEventSelectors"
      ],
      "Resource": "*"
    }
  ]
}
```

##### Policy 10 — `Deny-ProductionResourcesInNonprod` (Nonproduction OU)

Blocks creation of production-grade compute and database resources in development and staging accounts. Prevents accidentally launching expensive resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyProductionInstanceTypes",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "ForAnyValue:StringLike": {
          "ec2:InstanceType": [
            "m5d.*",
            "m5n.*",
            "m5dn.*",
            "c5d.*",
            "c5n.*",
            "r5d.*",
            "r5n.*",
            "r5dn.*",
            "p3.*",
            "p4.*",
            "g4dn.*",
            "g5.*",
            "x1e.*",
            "i3en.*",
            "d2.*",
            "h1.*"
          ]
        }
      }
    },
    {
      "Sid": "DenyProductionRDS",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance",
        "rds:ModifyDBInstance"
      ],
      "Resource": "arn:aws:rds:*:*:db:*",
      "Condition": {
        "ForAnyValue:StringLike": {
          "rds:DatabaseClass": [
            "db.r5.*",
            "db.r6g.*",
            "db.m5.*",
            "db.m6g.*",
            "db.x1e.*",
            "db.x2g.*"
          ]
        }
      }
    },
    {
      "Sid": "DenyMultiAZ",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance",
        "rds:ModifyDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "rds:MultiAZ": "true"
        }
      }
    }
  ]
}
```

##### Policy 11 — `Deny-PersistentResourcesInSandbox` (Sandbox OU)

Prevents creation of permanent data services (RDS, Secrets Manager secrets) and enforces ephemeral-only usage patterns in sandbox accounts.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRDSInSandbox",
      "Effect": "Deny",
      "Action": "rds:*",
      "Resource": "*"
    },
    {
      "Sid": "DenySecretsManagerCreate",
      "Effect": "Deny",
      "Action": [
        "secretsmanager:CreateSecret",
        "secretsmanager:PutSecretValue",
        "secretsmanager:RestoreSecret"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyPersistentEC2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "Bool": {
          "ec2:AssociatePublicIpAddress": "true"
        }
      }
    },
    {
      "Sid": "DenyEC2VolumePreservation",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:volume/*",
      "Condition": {
        "Bool": {
          "ec2:DeleteOnTermination": "false"
        }
      }
    }
  ]
}
```

#### SCP Evaluation Order

SCPs are evaluated in the following order per AWS Organizations rules:

1. Root-level deny policies (policies 1–4) apply to all accounts
2. OU-specific deny policies (policies 7–11) apply to accounts in those OUs
3. An explicit deny from any policy overrides any allow
4. Break-glass role (policies where excluded) bypasses OU boundary SCPs but still subject to global SCPs

> **Important**: These OU-specific SCPs are documented state. See `scp-implementation.md` for Terraform, AWS CLI, and Console deployment instructions.

## Break-Glass Procedure

In the event of an IAM Identity Center outage or loss of administrative access:

1. A designated break-glass IAM user exists in each account with a physically secured password
2. The password is stored in a fireproof safe (printed + sealed envelope)
3. Usage triggers an immediate SNS alert to CISO + IAM Lead
4. After resolution, the password is rotated and re-sealed
