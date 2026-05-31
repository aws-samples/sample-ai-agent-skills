---
title: "G2 — SMTP Credentials"
description: "Diagnose SES SMTP authentication issues"
status: active
severity: HIGH
triggers:
  - "SMTP authentication failed"
  - "SMTP credentials not working"
  - "535 Authentication failed"
  - "SMTP connection error"
owner: devops-agent
objective: "Resolve SES SMTP credential and connection issues"
context: "SES SMTP credentials are derived from IAM access keys using a specific algorithm. They are NOT the same as IAM credentials. SMTP endpoint is email-smtp.<region>.amazonaws.com on port 587 (STARTTLS) or 465 (TLS Wrapper)."
---

## Phase 1 — Triage

MUST:
- Verify SMTP endpoint and port: `email-smtp.<region>.amazonaws.com` port 587 or 465
- Confirm SMTP credentials were generated correctly (not raw IAM keys)
- Check IAM user has ses:SendRawEmail permission: `aws iam list-attached-user-policies --user-name <user>`
- Verify the IAM access key is active: `aws iam list-access-keys --user-name <user>`

SHOULD:
- Test SMTP connection with openssl: `openssl s_client -connect email-smtp.<region>.amazonaws.com:465`
- Verify TLS is being used (SES requires TLS)
- Check if the IAM user's access key was rotated (invalidates SMTP password)

MAY:
- Check network connectivity to SMTP endpoint (port 587/465)
- Verify no firewall blocking outbound SMTP traffic

## Phase 2 — Remediate

MUST:
- Generate SMTP credentials correctly via SES console or documented algorithm
- Do NOT use IAM access key/secret directly as SMTP credentials
- Ensure IAM user has ses:SendRawEmail permission

SHOULD:
- Use port 587 with STARTTLS (preferred) or port 465 with TLS Wrapper
- Regenerate SMTP credentials if IAM access key was rotated
- Store SMTP credentials securely (Secrets Manager or Parameter Store)

MAY:
- Consider using SES API (SendEmail/SendRawEmail) instead of SMTP for programmatic sending
- Set up dedicated IAM user for SMTP access

## Common Issues

- symptoms: "535 Authentication Credentials Invalid"
  diagnosis: "Using IAM access key/secret directly instead of derived SMTP password."
  resolution: "Generate SMTP credentials via SES console. SMTP password ≠ IAM secret key."

- symptoms: "SMTP connection timeout"
  diagnosis: "Firewall blocking port 587/465 or wrong SMTP endpoint."
  resolution: "Verify endpoint: email-smtp.<region>.amazonaws.com. Open port 587 or 465."

- symptoms: "SMTP worked before but stopped"
  diagnosis: "IAM access key was rotated, invalidating the derived SMTP password."
  resolution: "Regenerate SMTP credentials from the new IAM access key."

## Output Format

```yaml
root_cause: "smtp_credentials — <specific_cause>"
evidence:
  - type: smtp_config
    content: "<SMTP endpoint, port, and TLS configuration>"
  - type: iam_key_status
    content: "<IAM access key status>"
severity: HIGH
mitigation:
  immediate: "Regenerate SMTP credentials correctly"
  long_term: "Store credentials in Secrets Manager with rotation"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Email content may contain customer PII and business communications
- Recipient lists contain personal email addresses
- Bounce and complaint data reveals delivery issues with specific addresses
- Classification: HIGHLY CONFIDENTIAL - email content and recipient PII

## Prohibited Actions

- NEVER send test emails to real customer addresses
- NEVER modify production DKIM/SPF/DMARC records without email team approval
- NEVER disable bounce or complaint handling in production
- NEVER remove addresses from the suppression list without investigating the original cause

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling DKIM/SPF/DMARC"
  - "NEVER suggest removing sending limits"
  - "NEVER suggest disabling bounce/complaint handling"
