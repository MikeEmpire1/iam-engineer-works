# 01 — User Lifecycle (Joiner / Mover / Leaver)

## Scenario

The IAM team handles three events on Monday morning:

- **Joiner** — Daniel Park, new Platform Engineer, needs access to Nonproduction
- **Mover** — Maya Johnson transfers from App Dev to Platform Engineering
- **Leaver** — Kevin Nguyen resigned; all access must be revoked

## Key Files

| File | Description |
|---|---|
| `scenario.md` | Business narrative, requirements, success criteria |
| `design.md` | Identity source decision tree, joiner/mover/leaver process flows, group/permission set design, compliance mapping (Cyber Essentials Plus, ISO 27001) |
| `implementation.md` | Terraform configs, AWS CLI commands, PowerShell automation script (`process-lifecycle-events.ps1`), CloudTrail audit queries, HR reconciliation |

## Key Outcomes

- Daniel Park provisioned via Terraform with `engineering` + `platform-engineers` groups and `DevAccess` permission set on `inno-nonprod`
- Maya Johnson's group membership migrated from `app-dev` to `platform-engineers` with zero access interruption
- Kevin Nguyen deactivated: all groups removed, user disabled in IAM Identity Centre, CISO notified
- All events logged with HR ticket references (`HR-2026-071` through `HR-2026-069`)
