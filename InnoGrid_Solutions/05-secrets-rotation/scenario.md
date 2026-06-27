# Scenario 5: Secrets Rotation

## Background

InnoGrid's SOC 2 Type II audit revealed a finding: **IAM access keys and database passwords are not rotated on a regular schedule**. Several service accounts have keys that are over 2 years old. The RDS databases use static passwords that have never been rotated.

The CISO (Sarah Chen) has mandated automated rotation for all secrets:

| Secret Type | Current State | Rotation Requirement |
|---|---|---|
| IAM user access keys | Manual, no rotation | Rotate every 90 days |
| RDS database passwords | Static, never rotated | Rotate every 180 days |
| Third-party API keys | Hardcoded in config files | Rotate every 30 days |
| Break-glass IAM passwords | Static, in safe | Rotate every 90 days |
| X.509 certificates | Manual renewal | Rotate before expiry (30-day warning) |

## Situation

The IAM team has identified the following secrets that need immediate rotation automation:

### Service Account IAM Users

| Username | Purpose | Keys Age | Account |
|---|---|---|---|
| `svc-cicd-deploy` | CI/CD pipeline deployments | 487 days | Nonproduction |
| `svc-backup-agent` | Backup automation | 321 days | Production |
| `svc-monitoring` | CloudWatch metric publishing | 562 days | All accounts |
| `svc-sync-workday` | Workday HR data sync | 203 days | Management |

### RDS Instances

| DB Name | Engine | Account | Password Last Rotated |
|---|---|---|---|
| `innodb-prod-app` | PostgreSQL 15 | Production | Never (created 2023) |
| `innodb-nonprod-app` | PostgreSQL 15 | Nonproduction | Never (created 2024) |

### Break-Glass Credentials

| Account | User | Last Rotation |
|---|---|---|
| Management | `break-glass-admin` | 2024-01-15 (>500 days) |
| Production | `break-glass-prod` | 2024-01-15 (>500 days) |
| Security | `break-glass-sec` | 2024-01-15 (>500 days) |
| Nonproduction | `break-glass-nonprod` | 2024-01-15 (>500 days) |
| Sandbox | `break-glass-sandbox` | 2024-01-15 (>500 days) |

## Requirements

1. **Automated rotation** — All secrets must rotate automatically on schedule without manual intervention
2. **No downtime** — Rotation must not cause application or pipeline failures (key rotation uses two-key strategy)
3. **Audit trail** — Every rotation logged to CloudTrail with before/after hash verification
4. **Rollback** — If rotation fails, the previous secret must remain valid (staging strategy)
5. **Notifications** — Rotation failures alert IAM team via SNS → PagerDuty
6. **Break-glass rotation** — Break-glass passwords rotated every 90 days; new password stored in Secrets Manager (not just the safe)

## Success Criteria

- All 4 IAM service account keys rotated to <90 days old
- Both RDS passwords rotated with zero application downtime
- All 5 break-glass passwords rotated and stored in Secrets Manager
- Rotation history queryable via CloudTrail
- Failed rotation triggers a PagerDuty alert
