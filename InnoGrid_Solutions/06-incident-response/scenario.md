# Scenario 6: IAM Incident Response

## Background

InnoGrid's SOC (Tanya Brooks, Jake Hoffman, Olivia Reed) monitors AWS security 24/7 using GuardDuty, Security Hub, and CloudTrail. The IAM team (Ryan Mitchell, Aisha Patel, Miguel Torres) handles IAM-specific escalations.

The incident response plan follows the **NIST 800-61** framework:

1. **Preparation** — Runbooks, tools, and training in place
2. **Detection & Analysis** — Identify and validate the incident
3. **Containment, Eradication & Recovery** — Stop the bleeding, remove the threat, restore normal operations
4. **Post-Incident Activity** — Root cause analysis, lessons learned, control improvements

## Incident Alert

**Time: 2026-07-15 14:32 UTC**

The SOC receives a **GuardDuty finding**:

**Finding Type**: `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS`

**Severity**: **HIGH (7.0)**

**Details**:
- **User**: `svc-cicd-deploy` (IAM user in Nonproduction account)
- **Action**: `RunInstances` in `eu-central-1` (Frankfurt) — a region NOT in InnoGrid's allowed regions list (SCP restricts to us-east-1, us-west-2, eu-west-1)
- **IP**: `185.220.101.x` (Tor exit node)
- **Resource**: EC2 instance `i-0abcd1234efgh5678` launched at 14:28 UTC
- **Instance type**: `p3.16xlarge` (GPU instance — not in allowed types list)

### Additional Findings

| Time | Finding | Severity |
|---|---|---|
| 14:28 UTC | Instance `i-0abcd1234efgh5678` launched in eu-central-1 by `svc-cicd-deploy` | MEDIUM |
| 14:29 UTC | `svc-cicd-deploy` called `iam:CreateAccessKey` for user `svc-backup-agent` | HIGH |
| 14:30 UTC | Newly created key used to call `s3:ListBuckets` and `s3:GetObject` on `inno-prod-backup-data` | MEDIUM |
| 14:31 UTC | `svc-cicd-deploy` attempted `iam:CreateUser` — **denied** by SCP | LOW |
| 14:31 UTC | Data transfer of ~500MB from `inno-prod-backup-data` to external IP | HIGH |

### Impact Assessment

| Asset | Impact | Details |
|---|---|---|
| `svc-cicd-deploy` IAM user | **Compromised** | Access keys exfiltrated to attacker |
| `svc-backup-agent` IAM user | **Privilege escalation** | New keys created by attacker |
| `inno-prod-backup-data` S3 bucket | **Data exfiltration** | ~500MB of backup data downloaded |
| EC2 GPU instance | **Cryptomining** | `p3.16xlarge` launched in unauthorized region |
| IAM Identity Center | **Unaffected** | Incident limited to IAM users, not Identity Center |

## Response Objectives

1. **CONTAIN** — Stop the active attack within 15 minutes of detection
2. **ERADICATE** — Remove attacker access and compromised resources
3. **RECOVER** — Restore affected services and data access
4. **ANALYZE** — Determine root cause and scope of the breach
5. **IMPROVE** — Implement controls to prevent recurrence

## Success Criteria

- Active attacker sessions terminated within 15 minutes
- Compromised keys revoked within 15 minutes
- Rogue EC2 instance terminated within 15 minutes
- IAM users `svc-cicd-deploy` and `svc-backup-agent` secured (keys rotated)
- Root cause identified and documented
- At least 3 control improvements implemented post-incident
