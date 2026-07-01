# 04 — Federation / SSO Setup

## Status

**Complete**

## Scenario

Corporate users (HR, Finance, Legal, Exec) are manually duplicated across Entra ID and IAM Identity Centre. The project configures Entra ID as an external IdP with SAML federation and SCIM provisioning, enabling automatic user sync and SSO via the Entra ID My Apps portal.

## Key Files

| File | Description |
|---|---|
| `scenario.md` | Business case, requirements, success criteria (including 15-min SCIM sync) |
| `design.md` | Architecture diagram, SAML auth flow, SCIM sequence diagram, attribute mapping, conditional access policies, group sync design |
| `implementation.md` | IdP metadata exchange, SAML app config, SCIM endpoint setup, 4 conditional access policies, test procedures (SSO, SCIM deprovisioning, Engineering fallback), monitoring queries |

## Key Outcomes

- Entra ID configured as external IdP with SAML 2.0 federation
- SCIM provisioning enabled — users/groups sync automatically within ~40 minutes
- 3 conditional access policies enforced (MFA, block legacy auth, device compliance)
- Engineering users unaffected — direct IAM Identity Centre auth preserved as fallback
