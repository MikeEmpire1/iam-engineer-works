# 02 — Access Certification / Recertification

## Status

**Complete**

## Scenario

The CISO kicks off the Q3 2026 access review campaign. 11 managers review their 19 direct reports' access across IAM Identity Center and Entra ID. Miguel Torres has Sandbox access removed, Chris Evans loses Nonproduction access, and Grace Kim's access is suspended during parental leave.

## Key Files

| File | Description |
|---|---|
| `scenario.md` | Review campaign assignments, expected decisions, success criteria |
| `design.md` | Review lifecycle flow, CSV manifest schema, Entra ID Access Review integration, compliance mapping |
| `implementation.md` | Python manifest generator, PowerShell review distribution, decision processor, auto-revocation, Lambda restore script, compliance report generator, CloudTrail audit query |

## Key Outcomes

- 19 users recertified, 100% completion rate
- 2 modifications applied (Miguel Torres, Chris Evans)
- 1 suspension with automated restoration (Grace Kim)
- Compliance report generated for SOC 2 + ISO 27001 audit
