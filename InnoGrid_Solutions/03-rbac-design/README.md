# 03 — RBAC Design & Implementation

## Status

**Not yet implemented** — scenario template is in place.

## Planned Scope

- Define RBAC groups and permission sets for Engineering, IT, and cross-functional roles
- Map job functions to IAM Identity Center groups with least-privilege permission sets
- Implement Terraform modules for reusable group/permission-set definitions
- Design for separation of duties: no single group grants both read and write to production
- Just-in-time elevation workflow for production administrative access
