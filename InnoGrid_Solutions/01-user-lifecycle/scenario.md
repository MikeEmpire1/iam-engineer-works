# Scenario 1: User Lifecycle (Joiner / Mover / Leaver)

## Background

InnoGrid Solutions runs a hybrid identity model:

- **Engineering & IT** — managed directly in **AWS IAM Identity Center** (the source of truth for their identities and group memberships)
- **Corporate staff** (HR, Finance, Legal, Exec) — sourced from **Microsoft Entra ID**, synced via SCIM into IAM Identity Center for AWS access
- **Guests** (Sales contractors) — no AWS access by default

IAM Identity Center is the single plane of glass for all AWS authentication. Group membership determines which accounts and permission sets a user can access.

## Situation: Monday Morning

It's 9:00 AM on a Monday. The IAM team (Ryan Mitchell, Aisha Patel, Miguel Torres) finds three requests waiting:

### 1. Joiner — Daniel Park (Engineering)

HR sent a new-hire notification:

| Field | Value |
|---|---|
| **Name** | Daniel Park |
| **Title** | Platform Engineer |
| **Department** | Engineering (Platform) |
| **Manager** | Priya Sharma |
| **Level** | IC3 |
| **Identity Source** | AWS IAM Identity Center (direct) |
| **Start Date** | Today |
| **Access Needed** | Nonproduction account (`inno-nonprod`) — dev access |

Daniel is backfilling the role left by **Ian Scott** (terminated 2026-05-15).

### 2. Mover — Maya Johnson (Engineering → Engineering)

Derek Jones (App Dev Manager) emailed: Maya Johnson is transferring from his team to Priya Sharma's Platform Engineering team. Effective immediately.

| Field | Current | New |
|---|---|---|
| **Name** | Maya Johnson | Maya Johnson |
| **Title** | Application Developer | Platform Engineer |
| **Department** | Engineering (App Dev) | Engineering (Platform) |
| **Manager** | Derek Jones | Priya Sharma |
| **Level** | IC3 | IC3 (no change) |
| **Identity Source** | IAM Identity Center | IAM Identity Center |
| **Access Change** | Remove `app-dev` group | Add `platform-engineers` group |

### 3. Leaver — Kevin Nguyen (IT & Security)

HR processed a separation notice: **Kevin Nguyen** (Helpdesk Technician, IC1) resigned effective last Friday.

| Field | Value |
|---|---|
| **Name** | Kevin Nguyen |
| **Title** | Helpdesk Technician |
| **Department** | IT & Security (Helpdesk) |
| **Manager** | Carlos Mendez |
| **Level** | IC1 |
| **Identity Source** | AWS IAM Identity Center (direct) |
| **Last Day** | Last Friday |
| **Action Required** | Full deprovisioning |

## Requirements

1. **Joiner** — Provision a new user in IAM Identity Center, assign to the correct groups, and grant scoped access to the Nonproduction account within 4 hours of notification.
2. **Mover** — Update group memberships and manager attribute. Revoke old group access and grant new group access with zero unintended downtime.
3. **Leaver** — Disable the user account within 1 hour of notification. Remove all group memberships and active sessions. Archive home directory and audit logs before deletion (retention period: 90 days).
4. **Audit** — Every lifecycle event must produce a log entry (CloudTrail, IAM Identity Center events) that can be traced back to the requesting HR ticket number.
5. **Compliance** — All changes must comply with SOC 2 and ISO 27001 access control requirements (documented change, approval, verification).

## Success Criteria

- Daniel Park can sign in to the AWS access portal and access the Nonproduction account with the `DevAccess` permission set within 4 hours.
- Maya Johnson's App Dev group membership is removed and she can access Platform Engineering resources without interruption to her existing Nonproduction access (permission set stays the same; only group membership changes).
- Kevin Nguyen cannot sign in to the AWS access portal. His IAM Identity Center user is disabled and all group memberships removed. A termination confirmation is sent to the CISO.
- All three events are logged with HR ticket references and are queryable via CloudTrail.
