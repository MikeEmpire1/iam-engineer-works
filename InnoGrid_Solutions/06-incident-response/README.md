# 06 — IAM Incident Response

## Status

**Complete**

## Scenario

A GuardDuty HIGH severity finding detects `svc-cicd-deploy` (CI/CD service account) launching a GPU instance in an unauthorised region from a Tor exit node. The attacker escalates privileges, creates keys for another service account, and exfiltrates ~500MB of backup data from S3.

## Key Files

| File | Description |
|---|---|
| `scenario.md` | Full incident timeline, GuardDuty findings, impact assessment, response objectives |
| `design.md` | IR architecture, severity levels (SEV-1/2/3), playbook flow, containment options, forensic data sources, compliance mapping |
| `implementation.md` | Complete response runbook: key deactivation, instance termination (with forensic snapshot), emergency SCP, secrets rotation, CloudTrail/GuardDuty investigation, root cause analysis, 6 control improvements (SCP, IAM Access Analyser, Tor threat list, auto-contain Lambda, EventBridge rule), post-incident report |

## Key Outcomes

- **Time-to-contain: 13 minutes** (under 15-min SEV-1 SLA)
- Compromised keys deactivated, rogue instance terminated, emergency SCP applied
- Root cause identified: Jenkins CI/CD with no MFA + plaintext AWS keys
- 6 control improvements implemented post-incident
