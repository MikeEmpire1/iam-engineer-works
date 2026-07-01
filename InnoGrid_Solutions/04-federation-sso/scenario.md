# Scenario 4: Federation / SSO Setup

## Background

InnoGrid operates a **hybrid identity model**:

- **Engineering & IT** (~160 users) managed directly in AWS IAM Identity Centre
- **Corporate staff** (~140 users) sourced from Microsoft Entra ID

Currently, corporate users are created manually in both Entra ID and IAM Identity Centre — a duplicate effort that leads to drift. The CTO (Elena Vasquez) has approved a project to:

1. Configure **Entra ID as an external identity provider** for AWS IAM Identity Centre
2. Enable **SCIM provisioning** so Entra ID users and groups sync automatically
3. Enforce **conditional access policies** (MFA, device compliance, trusted locations)
4. Provide a **single sign-on (SSO)** experience: users log in via Entra ID → see their AWS apps in the My Apps portal → click through to AWS

## Requirements

1. **Federation** — Corporate users authenticate via Entra ID; Engineering users continue to authenticate directly in IAM Identity Centre
2. **SCIM provisioning** — Entra ID pushes user and group updates to IAM Identity Centre automatically (create, update, suspend, delete)
3. **SSO portal** — Corporate users access AWS via the Entra ID My Apps portal or the AWS access portal (same experience)
4. **Conditional access** — MFA required for all AWS access; device compliance required for production; trusted locations for admin elevation
5. **Attribute mapping** — Entra ID attributes (department, manager, employee ID) map to IAM Identity Centre attributes for RBAC
6. **Fallback** — If Entra ID is unavailable, Engineering users can still authenticate directly via IAM Identity Centre

## Success Criteria

- A corporate user (e.g., Diana Cruz, Finance) signs into the Entra ID My Apps portal → clicks AWS → is logged into the AWS access portal → can access `Prod-ReadOnly` in the Production account
- A new corporate user created in Entra ID appears in IAM Identity Centre within 15 minutes (SCIM sync interval)
- A corporate user disabled in Entra ID is suspended in IAM Identity Centre within 15 minutes
- Conditional access blocks AWS access from non-corporate devices or untrusted locations
- Engineering users still authenticate directly via IAM Identity Centre (no disruption)
