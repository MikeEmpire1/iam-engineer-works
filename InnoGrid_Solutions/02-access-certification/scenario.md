# Scenario 2: Access Certification / Recertification

## Background

InnoGrid's Cyber Essentials Plus and ISO 27001 certifications require **quarterly access reviews**. Every manager must recertify their direct reports' access to AWS accounts at least once per quarter. Access that is not recertified within the review window is automatically revoked.

The review covers:

- **IAM Identity Centre users** (Engineering & IT) — group memberships and permission set assignments
- **Entra ID synced users** (Corporate) — group memberships in IAM Identity Centre
- **Permission set assignments** — which accounts each group can access

## Situation: Q3 2026 Access Review

The CISO (Sarah Chen) kicks off the Q3 2026 access certification campaign. The review window is **July 1 – July 14, 2026**.

### Review Workflow

```
CISO opens campaign
       │
       ▼
Assign reviewers (managers review their direct reports)
       │
       ▼
Managers log in → review access → Approve / Revoke / Modify
       │
       ▼
End of window → auto-revoke any unrecertified access
       │
       ▼
Generate compliance report for auditors
```

### Sample Review Assignments

| Reviewer | Reviews | # of Reports |
|---|---|---|
| Priya Sharma | Platform Engineers (Alex Rivera, Sam Green, Daniel Park) | 3 |
| Derek Jones | App Developers (Ethan Brown) — Maya Johnson transferred out | 1 |
| Lisa Kim | QA Engineers (Chloe Wilson) | 1 |
| Ryan Mitchell | IAM Engineers (Aisha Patel, Miguel Torres) | 2 |
| Tanya Brooks | SOC Analysts (Jake Hoffman, Olivia Reed) | 2 |
| Carlos Mendez | Service Desk (Emily Zhao) — Kevin Nguyen terminated | 1 |
| Rebecca Torres | HRIS (Amanda Foster), Recruiter (Jordan Bell) | 2 |
| Nathan Cole | Accountants (Diana Cruz, Grace Kim — on leave) | 2 |
| Peter Griffin | Contracts Specialist (Sarah Huang) | 1 |
| Rachel Adams | Marketing (Nina Patel, Chris Evans), Account Exec (Tom Watson) | 3 |
| Angela Wright | Operations (Ben Schneider, Maria Gomez) | 2 |

### Expected Review Decisions

| Employee | Reviewer | Expected Decision | Rationale |
|---|---|---|---|
| Alex Rivera | Priya Sharma | **Approve** | Senior engineer, active |
| Sam Green | Priya Sharma | **Approve** | Active, no changes |
| Daniel Park | Priya Sharma | **Approve** | New joiner, correct access |
| Ethan Brown | Derek Jones | **Approve** | Active developer |
| Chloe Wilson | Lisa Kim | **Approve** | Active QA engineer |
| Aisha Patel | Ryan Mitchell | **Approve** | Senior IAM engineer |
| Miguel Torres | Ryan Mitchell | **Modify** | Remove Sandbox access (no longer needed for project) |
| Jake Hoffman | Tanya Brooks | **Approve** | Active SOC senior |
| Olivia Reed | Tanya Brooks | **Approve** | Active SOC analyst |
| Emily Zhao | Carlos Mendez | **Approve** | Active service desk tech |
| Amanda Foster | Rebecca Torres | **Approve** | Active HRIS specialist |
| Jordan Bell | Rebecca Torres | **Approve** | Active recruiter |
| Diana Cruz | Nathan Cole | **Approve** | Active accountant |
| Grace Kim | Nathan Cole | **Revoke (temporary)** | On leave until 2026-08-01; access suspended during leave |
| Sarah Huang | Peter Griffin | **Approve** | Active contracts specialist |
| Nina Patel | Rachel Adams | **Approve** | Active marketing manager |
| Chris Evans | Rachel Adams | **Modify** | Remove Nonproduction access (no longer needs dev environment) |
| Tom Watson | Rachel Adams | **Approve** | Guest account, limited access |
| Ben Schneider | Angela Wright | **Approve** | Active ops manager |
| Maria Gomez | Angela Wright | **Approve** | Active admin coordinator |

## Requirements

1. **Campaign setup** — Create a Q3 2026 access review campaign covering all IAM Identity Centre users with AWS access
2. **Reviewer assignment** — Map each user to their manager as the reviewer
3. **Recertification** — Managers review and either approve, modify, or revoke access for each direct report
4. **Auto-revocation** — After the July 14 deadline, any unrecertified access is automatically revoked
5. **Audit evidence** — Generate a compliance report showing who reviewed what, when, and their decision
6. **Temporary suspensions** — Handle employees on leave (Grace Kim) by suspending access during leave, restoring on return
7. **Incidental changes** — Miguel Torres (remove Sandbox), Chris Evans (remove Nonproduction)

## Success Criteria

- All 19 active users are reviewed and recertified or modified by their managers
- Grace Kim's access is suspended and restored on return
- Miguel Torres's Sandbox access is removed; Chris Evans's Nonproduction access is removed
- A compliance report is generated showing 100% recertification with audit trail
- CloudTrail logs capture all permission set and group membership changes from the campaign
