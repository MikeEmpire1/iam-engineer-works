# InnoGrid Solutions — Company Profile

## Overview

| Attribute | Detail |
|---|---|
| **Company Name** | InnoGrid Solutions |
| **Headquarters** | London, UK |
| **Founded** | 2018 |
| **Industry** | Cloud Consulting & Managed Services |
| **Employees** | ~300 (growing) |
| **Annual Revenue** | ~£35M (2025) |
| **Primary Cloud Provider** | AWS |
| **Secondary Identity Provider** | Microsoft Entra ID (for corporate identity) |

## Mission

InnoGrid Solutions helps enterprises accelerate their cloud journey through architecture consulting, migration services, managed cloud operations, and DevSecOps enablement. Clients range from Series B startups to FTSE 350 companies across healthcare, finance, and e-commerce.

## Service Lines

1. **Cloud Architecture & Migration** — Lift-and-shift, re-platform, and re-architecture engagements
2. **Managed Cloud Operations** — 24/7 monitoring, incident response, cost optimisation
3. **DevSecOps & Platform Engineering** — CI/CD pipelines, security automation, internal developer platforms
4. **IAM & Security Consulting** — Identity governance, access certification, federation, secrets management

## Department Overview

| Department | Head Count | Identity Source |
|---|---|---|
| Engineering (Platform, App Dev, QA) | ~120 | AWS IAM Identity Centre |
| IT & Security (IAM, SOC, Service Desk) | ~40 | AWS IAM Identity Centre |
| HR | ~15 | Entra ID |
| Finance | ~12 | Entra ID |
| Legal | ~8 | Entra ID |
| Sales & Marketing | ~65 | Guest / Entra ID |
| Executive | ~6 | Entra ID |
| Operations & Admin | ~35 | Entra ID |

## Regulatory & Compliance Context

InnoGrid holds or is pursuing:

- **Cyber Essentials Plus** — Annual certification (2025, 2026)
- **ISO 27001** — Certified
- **UK Data Protection Act 2018** — Compliance for UK-based clients and data subjects
- **UK GDPR** — Compliance for UK-based data processing

These certifications impose strict identity lifecycle, access review, and secrets rotation requirements — directly driving the need for robust IAM practices.

## Access Model Philosophy

InnoGrid follows **least-privilege** with **just-in-time (JIT)** access for production. Identity sources are split:

- **Entra ID** is the system of record for HR, Finance, Legal, and Exec — synced via SCIM to AWS IAM Identity Centre for AWS access.
- **AWS IAM Identity Centre** directly manages Engineering and IT identities, with group-based permission sets for role-based access.

## Tooling Stack

| Capability | Tool |
|---|---|
| Identity Provider (Corporate) | Microsoft Entra ID (P2) |
| AWS SSO / Identity Centre | AWS IAM Identity Centre |
| Infrastructure as Code | Terraform (HashiCorp Cloud Platform backend) |
| Secret Management | AWS Secrets Manager |
| Access Certification | Entra ID Access Reviews + custom AWS review scripts |
| Monitoring & Detection | AWS CloudTrail, Amazon GuardDuty, Security Hub |
| Incident Response | PagerDuty + Jira Service Management |
| Passwordless Auth | FIDO2 security keys (Engineering), TOTP (all others) |
| HR Sync | Workday (exported to CSV, ingested via custom Lambda) |
