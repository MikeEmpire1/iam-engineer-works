# 05 — Secrets Rotation

## Status

**Complete**

## Scenario

A Cyber Essentials Plus audit finding reveals IAM keys over 500 days old, RDS passwords never rotated, and break-glass credentials unchanged for 18 months. The IAM team implements Lambda-backed automated rotation for all secrets via AWS Secrets Manager.

## Key Files

| File | Description |
|---|---|
| `scenario.md` | Audit finding details (4 service accounts, 2 RDS instances, 5 break-glass users), requirements, success criteria |
| `design.md` | Secrets Manager architecture, 3 rotation strategies (two-key IAM, staged RDS, direct break-glass), rotation schedules, Lambda IAM permissions, compliance mapping |
| `implementation.md` | Terraform Secrets Manager + KMS, 3 Lambda rotation functions (IAM keys, RDS password, break-glass), initial secret population scripts, rotation enablement, verification queries, manual console alternatives |

## Key Outcomes

- 4 IAM service account keys rotated on 90-day schedule (zero-downtime two-key strategy)
- 2 RDS passwords rotated on 180-day schedule
- 5 break-glass passwords rotated on 90-day schedule with CISO notification
- Rotation failures alert via SNS → PagerDuty
