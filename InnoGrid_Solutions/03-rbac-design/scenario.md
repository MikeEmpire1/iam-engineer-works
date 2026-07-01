# Scenario 3: RBAC Design & Implementation

## Background

InnoGrid's IAM program started ad-hoc — groups and permission sets were created on demand without a consistent naming convention or role hierarchy. As the company grew to 300 employees and 5 AWS accounts, this became unmanageable.

The CISO (Sarah Chen) has mandated a formal **RBAC (Role-Based Access Control)** redesign:

- Every group must map to a business function, not an individual
- Permission sets must follow least privilege
- Production access requires just-in-time (JIT) elevation
- Naming conventions must be standardised across all accounts

## Current State (Problem)

| Issue | Example |
|---|---|
| Groups named after individuals | `alex-rivera-admin`, `priya-sharma-sandbox` |
| Duplicate permission sets | `DevAccess`, `dev-access`, `DeveloperAccess` — all identical |
| Standing prod access | Engineers have `AdminAccess` on production permanently |
| No naming convention | `eng-group`, `Engineering-Team`, `Engineers-2026` |
| Orphaned groups | `old-contractors-2024` — 20 members, all departed |

## Requirements

1. **Role taxonomy** — Define a clear hierarchy of roles: Job Function → Role → Permission Set
2. **Naming convention** — Standardise group and permission set names across all 5 accounts
3. **Least privilege** — No permission set grants more access than necessary for the role
4. **JIT for production** — Engineers get standing read-only access to production; write access requires JIT elevation with approval
5. **Terraform modules** — Reusable, version-controlled RBAC definitions
6. **Separation of duties** — No single role allows both provisioning and using resources in production
7. **Migration** — Plan to migrate from current ad-hoc groups to the new RBAC structure without downtime

## Success Criteria

- All 50+ groups consolidated into ≤15 RBAC-aligned groups
- Permission sets standardised to 5 distinct tiers
- Production admin access requires JIT elevation (no standing grants)
- Terraform modules used to deploy all RBAC resources
- Migration completed with zero unplanned downtime
