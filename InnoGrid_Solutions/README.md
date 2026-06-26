# InnoGrid Solutions — IAM Engineering Portfolio

This directory contains the complete IAM engineering portfolio for InnoGrid Solutions, a fictional cloud consulting firm. Every scenario follows a real-world workflow: understand the business context, design the solution, implement with IaC (Terraform) + scripts, and document evidence.

## Directory Layout

| Path | Description |
|---|---|
| `00-company-context/` | Foundational company documents: profile, org chart, personnel roster, AWS accounts |
| `01-user-lifecycle/` | Joiner/mover/leaver identity lifecycle automation |
| `02-access-certification/` | Periodic access review and recertification |
| `03-rbac-design/` | Role-based access control group/permission set design |
| `04-federation-sso/` | Entra ID ↔ AWS IAM Identity Center federation |
| `05-secrets-rotation/` | AWS Secrets Manager automated rotation |
| `06-incident-response/` | IAM incident detection, containment, and remediation |

## Key Design Principles

- **Least privilege** — JIT access for production, group-based permission sets
- **Separation of duties** — Identity source split: Entra ID (corporate), IAM Identity Center (engineering)
- **Audit-first** — Every operation logged to CloudTrail with HR ticket references
- **Compliance-driven** — SOC 2, ISO 27001, HIPAA controls inform every design decision
