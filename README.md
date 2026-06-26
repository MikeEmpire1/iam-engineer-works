# IAM Engineer Portfolio — InnoGrid Solutions

A hands-on IAM engineering portfolio demonstrating identity lifecycle, access certification, RBAC, federation/SSO, secrets rotation, and incident response in a realistic AWS + Entra ID hybrid environment.

## Structure

```
InnoGrid_Solutions/
├── 00-company-context/        Company profile, org chart, personnel roster, AWS accounts
├── 01-user-lifecycle/         Joiner / mover / leaver lifecycle
├── 02-access-certification/   Access reviews and recertification
├── 03-rbac-design/            Role-based access control design & implementation
├── 04-federation-sso/         Federation / SSO with Entra ID + IAM Identity Center
├── 05-secrets-rotation/       Automated secrets rotation with AWS Secrets Manager
└── 06-incident-response/      IAM incident detection and response
```

Each scenario contains:
- **scenario.md** — Business narrative and requirements
- **design.md** — IAM solution architecture
- **implementation.md** — Terraform, scripts, AWS CLI, and console workflows
- **evidence/** — Screenshots, logs, and verification artifacts

## Company Profile

**InnoGrid Solutions** is a fictional cloud consulting firm (~300 employees) with a hybrid identity model:
- **Engineering & IT** — managed directly in AWS IAM Identity Center
- **Corporate staff** (HR, Finance, Legal, Exec) — sourced from Entra ID, synced via SCIM
- **5 AWS accounts** across Management, Security, Infrastructure, and Sandbox OUs

## Scenarios

| # | Scenario | Status |
|---|---|---|
| 1 | User lifecycle (joiner/mover/leaver) | Complete |
| 2 | Access certification / recertification | Pending |
| 3 | RBAC design & implementation | Pending |
| 4 | Federation / SSO setup | Pending |
| 5 | Secrets rotation | Pending |
| 6 | IAM incident response | Pending |
