# Scenario 0: Foundation — IAM Identity Centre Initialization

## Overview

Before any user lifecycle events can occur, the IAM Identity Centre must be bootstrapped with groups, permission sets, users, and account assignments. This scenario establishes the foundation that all subsequent scenarios (1–6) build upon.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     AWS Organizations                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Management Account (1111-2222-3333)         │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │         AWS IAM Identity Centre                  │   │   │
│  │  │                                                  │   │   │
│  │  │  Users ←→ Groups ←→ Permission Sets ←→ Accounts  │   │   │
│  │  │                                                  │   │   │
│  │  │  Identity Store: d-9a7b8c6d5e                    │   │   │
│  │  │  Instance ARN: ssoins-1234567890abcdef            │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Entra ID (Corporate Identity)                │   │
│  │  HR, Finance, Legal, Exec users                          │   │
│  │  └── SCIM sync ──► IAM Identity Centre                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## Group Definitions

| Group Name | Display Name | Purpose | Member Count |
|---|---|---|---|
| `engineering` | Engineering | All engineers — grants baseline DevAccess to Nonproduction | ~9 |
| `platform-engineers` | Platform Engineers | Platform engineering team | ~4 |
| `app-dev` | Application Developers | Application development team | ~3 |
| `qa-engineers` | QA Engineers | QA & testing team | ~2 |
| `engineering-managers` | Engineering Managers | Engineering management (M2+ level) | ~4 |
| `it-security` | IT & Security | Broad IT/Security group — grants ReadOnly to Security account | ~7 |
| `iam-team` | IAM Team | IAM administration team — grants AdministratorAccess (JIT) | ~3 |
| `soc-team` | SOC Team | Security operations team — grants SecurityAudit | ~3 |
| `service-desk` | Service Desk | IT support / service desk team — grants ReadOnly | ~3 |
| `break-glass` | Break Glass | Emergency access group — grants AdministratorAccess | ~2 |

## Permission Set Definitions

| Permission Set | Managed Policies | Inline Policy | Session Duration | Relay State |
|---|---|---|---|---|
| `DevAccess` | `ReadOnlyAccess` | `s3:ListBucket, s3:GetObject` on `inno-nonprod-dev-artifacts` | 8h | EC2 console |
| `ReadOnly` | `ReadOnlyAccess` | — | 4h | AWS console home |
| `SecurityAudit` | `SecurityAudit` | — | 4h | Security Hub |
| `AdministratorAccess` | `AdministratorAccess` | — | 1h | AWS console home |
| `PowerUserAccess` | `PowerUserAccess` | — | 4h | AWS console home |

## User Provisioning Matrix

All active users with identity source `IAM Identity Centre` from the personnel roster:

| User | Title | Groups |
|---|---|---|
| James Okafor | VP of Engineering | `engineering`, `engineering-managers` |
| Priya Sharma | Platform Engineering Manager | `engineering`, `engineering-managers`, `platform-engineers` |
| Derek Jones | Application Development Manager | `engineering`, `engineering-managers`, `app-dev` |
| Lisa Kim | QA & Testing Manager | `engineering`, `engineering-managers`, `qa-engineers` |
| Ryan Mitchell | IAM Lead | `it-security`, `iam-team` |
| Aisha Patel | IAM Engineer (Senior) | `it-security`, `iam-team` |
| Miguel Torres | IAM Engineer | `it-security`, `iam-team` |
| Tanya Brooks | SOC Manager | `it-security`, `soc-team` |
| Jake Hoffman | SOC Analyst (Senior) | `it-security`, `soc-team` |
| Olivia Reed | SOC Analyst | `it-security`, `soc-team` |
| Carlos Mendez | IT Support Manager | `it-security`, `service-desk` |
| Emily Zhao | Service Desk Technician (Senior) | `it-security`, `service-desk` |
| Kevin Nguyen | Service Desk Technician | `it-security`, `service-desk` |
| Alex Rivera | Platform Engineer (Senior) | `engineering`, `platform-engineers` |
| Sam Green | Platform Engineer | `engineering`, `platform-engineers` |
| Maya Johnson | Application Developer | `engineering`, `app-dev` |
| Ethan Brown | Application Developer | `engineering`, `app-dev` |
| Chloe Wilson | QA Engineer | `engineering`, `qa-engineers` |
| Daniel Park | Platform Engineer (new) | `engineering`, `platform-engineers` (provisioned in Scenario 1) |

## Account Assignment Matrix

| Group | Permission Set | Target Account |
|---|---|---|
| `engineering` | `DevAccess` | Nonproduction (`123456789012`) |
| `platform-engineers` | `DevAccess` | Nonproduction (`123456789012`) |
| `app-dev` | `DevAccess` | Nonproduction (`123456789012`) |
| `qa-engineers` | `DevAccess` | Nonproduction (`123456789012`) |
| `it-security` | `ReadOnly` | Security (`444455556666`) |
| `iam-team` | `AdministratorAccess` | Management (`111122223333`) |
| `soc-team` | `SecurityAudit` | Security (`444455556666`) |
| `service-desk` | `ReadOnly` | Nonproduction (`123456789012`) |
| `break-glass` | `AdministratorAccess` | All accounts |

## SCIM Sync Architecture

```
┌──────────────┐     SAML 2.0     ┌──────────────────────┐
│   Entra ID   │ ◄──────────────► │  IAM Identity Centre │
│  (Corporate  │                  │  (AWS)               │
│   IdP)       │ ──────────────►  │                      │
│              │   SCIM v2.0      │  Users & Groups      │
│  HR, Finance │   (auto-sync)    │  synced from Entra   │
│  Legal, Exec │                  │                      │
└──────────────┘                  └──────────────────────┘
```

Corporate users (Entra ID source) are synced automatically via SCIM. Their group memberships are managed in Entra ID and propagated to IAM Identity Centre. Engineering & IT users are managed directly in IAM Identity Centre.

## Compliance Mapping

| Requirement | Control | How Its Met |
|---|---|---|
| ISO 27001 A.9.2.1 | User registration & de-registration | All users provisioned with documented group assignments |
| ISO 27001 A.9.2.3 | Management of privileged access | Privileged groups (`iam-team`, `break-glass`) have separate permission sets with short session durations |
| Cyber Essentials Plus | Access control | Group-based access with least-privilege permission sets |
| Cyber Essentials Plus | Multi-factor authentication | IAM Identity Centre enforces MFA at the portal level |
