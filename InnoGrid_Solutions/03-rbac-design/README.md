# 03 — RBAC Design & Implementation

## Status

**Complete**

## Scenario

InnoGrid's ad-hoc group structure (50+ groups, duplicate permission sets, orphaned groups) is replaced with a formal RBAC model. The new design standardises naming, enforces least privilege, and introduces JIT elevation for production admin access.

## Key Files

| File | Description |
|---|---|
| `scenario.md` | Current state problems, requirements, migration success criteria |
| `design.md` | 3-layer RBAC taxonomy (Job Function → Role → Permission Set), 16 standardised groups, 5 permission set tiers, JIT elevation workflow, separation of duties matrix, migration strategy |
| `implementation.md` | Terraform RBAC module, full config with all groups/permission sets, JIT Lambda, CloudWatch auto-expiry, PowerShell migration script (phase 1–5), verification commands |

## Key Outcomes

- 50+ ad-hoc groups consolidated to 16 standardised RBAC groups
- 5 distinct permission set tiers instead of 8+ duplicates
- Standing prod admin eliminated — JIT elevation with 1-hour auto-expiry
- `<function>-<role>` naming convention enforced
- All orphaned groups cleaned up
