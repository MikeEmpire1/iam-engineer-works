# InnoGrid Solutions — Company Profile

## Overview

| Attribute | Detail |
|---|---|
| **Company Name** | InnoGrid Solutions |
| **Headquarters** | Austin, Texas |
| **Founded** | 2018 |
| **Industry** | Cloud Consulting & Managed Services |
| **Employees** | ~300 (growing) |
| **Annual Revenue** | ~$45M (2025) |
| **Primary Cloud Provider** | AWS |
| **Secondary Identity Provider** | Microsoft Entra ID (for corporate identity) |

## Mission

InnoGrid Solutions helps enterprises accelerate their cloud journey through architecture consulting, migration services, managed cloud operations, and DevSecOps enablement. Clients range from Series B startups to Fortune 500 companies across healthcare, finance, and e-commerce.

## Service Lines

1. **Cloud Architecture & Migration** — Lift-and-shift, re-platform, and re-architecture engagements
2. **Managed Cloud Operations** — 24/7 monitoring, incident response, cost optimization
3. **DevSecOps & Platform Engineering** — CI/CD pipelines, security automation, internal developer platforms
4. **IAM & Security Consulting** — Identity governance, access certification, federation, secrets management

## Department Overview

| Department | Head Count | Identity Source |
|---|---|---|
| Engineering (Platform, App Dev, QA) | ~120 | AWS IAM Identity Center |
| IT & Security (IAM, SOC, Helpdesk) | ~40 | AWS IAM Identity Center |
| HR | ~15 | Entra ID |
| Finance | ~12 | Entra ID |
| Legal | ~8 | Entra ID |
| Sales & Marketing | ~65 | Guest / Entra ID |
| Executive | ~6 | Entra ID |
| Operations & Admin | ~35 | Entra ID |

## Regulatory & Compliance Context

InnoGrid holds or is pursuing:

- **SOC 2 Type II** — Annual audit (2025, 2026)
- **ISO 27001** — Certified
- **HIPAA** — Business Associate agreements with healthcare clients
- **GDPR** — Compliance for EU-based clients

These certifications impose strict identity lifecycle, access review, and secrets rotation requirements — directly driving the need for robust IAM practices.

## Access Model Philosophy

InnoGrid follows **least-privilege** with **just-in-time (JIT)** access for production. Identity sources are split:

- **Entra ID** is the system of record for HR, Finance, Legal, and Exec — synced via SCIM to AWS IAM Identity Center for AWS access.
- **AWS IAM Identity Center** directly manages Engineering and IT identities, with group-based permission sets for role-based access.

## Tooling Stack

| Capability | Tool |
|---|---|
| Identity Provider (Corporate) | Microsoft Entra ID (P2) |
| AWS SSO / Identity Center | AWS IAM Identity Center |
| Infrastructure as Code | Terraform (HashiCorp Cloud Platform backend) |
| Secret Management | AWS Secrets Manager |
| Access Certification | Entra ID Access Reviews + custom AWS review scripts |
| Monitoring & Detection | AWS CloudTrail, Amazon GuardDuty, Security Hub |
| Incident Response | PagerDuty + Jira Service Management |
| Passwordless Auth | FIDO2 security keys (Engineering), TOTP (all others) |
| HR Sync | Workday (exported to CSV, ingested via custom Lambda) |
