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
InnoGrid enforces the following SCP guardrails:

1. **Root activity block** — No root user actions except under break-glass procedure
2. **Region restriction** — Only `us-east-1`, `us-west-2`, `eu-west-1` allowed (compliance regions)
3. **No unapproved instance types** — Only `t3.*`, `m5.*`, `c5.*`, `r5.*` families allowed
4. **Require encryption** — EBS volumes, S3 buckets, and RDS instances must be encrypted
5. **No public S3 buckets** — `s3:PutBucketPublicAccessBlock` enforced
6. **JIT for production** — No standing `AdministratorAccess` in production account

## Break-Glass Procedure

In the event of an IAM Identity Center outage or loss of administrative access:

1. A designated break-glass IAM user exists in each account with a physically secured password
2. The password is stored in a fireproof safe (printed + sealed envelope)
3. Usage triggers an immediate SNS alert to CISO + IAM Lead
4. After resolution, the password is rotated and re-sealed
