---
title: "G3 — Shield Response Team (SRT) Engagement"
description: "Diagnose SRT access issues and guide proper SRT engagement"
status: active
severity: CRITICAL
triggers:
  - "SRT"
  - "Shield Response Team"
  - "DDoS support"
  - "Shield team"
  - "DDoS assistance"
owner: devops-agent
objective: "Ensure proper SRT engagement for DDoS attack response"
context: "The Shield Response Team (SRT) is a 24/7 team of AWS DDoS experts available to Shield Advanced subscribers. SRT can help with attack mitigation, WAF rule creation, and post-attack analysis. SRT engagement requires Shield Advanced subscription AND an IAM role granting SRT access to WAF and Shield resources. Proactive engagement allows SRT to contact you during detected attacks."
---

## Phase 1 — Triage

MUST:
- Verify Shield Advanced subscription is active: `aws shield describe-subscription`
- Check DRT access role: `aws shield describe-drt-access`
- Check emergency contact settings: `aws shield describe-emergency-contact-settings`
- Verify proactive engagement status: `aws shield describe-subscription --query 'Subscription.ProactiveEngagementStatus'`

SHOULD:
- Verify the DRT role has the required permissions (AWSShieldDRTAccessPolicy)
- Check if the DRT has access to WAF logs (S3 bucket access)
- Verify emergency contact information is current

MAY:
- Review past SRT engagements and recommendations
- Check if the DRT role trust policy allows the Shield service principal

## Phase 2 — Remediate

MUST:
- Create the DRT access role with AWSShieldDRTAccessPolicy: create an IAM role with `drt.shield.amazonaws.com` as the trusted service
- Associate the role: `aws shield associate-drt-role --role-arn <role-arn>`
- Grant DRT access to WAF log S3 buckets: `aws shield associate-drt-log-bucket --log-bucket <bucket-name>`
- Configure emergency contacts: `aws shield associate-proactive-engagement-details --emergency-contact-list EmailAddress=<email>,PhoneNumber=<phone>,ContactNotes=<notes>`

SHOULD:
- Enable proactive engagement: `aws shield enable-proactive-engagement`
- Ensure Route 53 health checks are configured (required for proactive engagement)
- Document the SRT engagement process for your team

MAY:
- Set up a runbook for SRT engagement during incidents
- Create a dedicated Slack/PagerDuty channel for SRT communications
- Pre-authorize SRT to make WAF changes during attacks

## Common Issues

- symptoms: "Cannot engage SRT — access denied"
  diagnosis: "Shield Advanced subscription is not active or DRT role is not configured."
  resolution: "Verify subscription. Create and associate the DRT access role."

- symptoms: "SRT cannot access WAF logs"
  diagnosis: "DRT log bucket access not granted."
  resolution: "Associate the WAF log S3 bucket with DRT access."

- symptoms: "SRT did not proactively contact during attack"
  diagnosis: "Proactive engagement not enabled, or no health check associated with the resource."
  resolution: "Enable proactive engagement. Associate health checks with all protected resources."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create DRT access role | YELLOW | IAM change; reversible |
| Associate DRT role | YELLOW | Configuration change; reversible |
| Grant DRT log access | YELLOW | Permission change; reversible |
| Enable proactive engagement | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Shield Advanced configuration changes
- DRT access role modifications
- Emergency contact changes
- Proactive engagement enablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-subscription` | LOW | Subscription status |
| `describe-drt-access` | MEDIUM | DRT role and log bucket access |
| `describe-emergency-contact-settings` | HIGH | Contact PII (email, phone) |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| DRT role creation | Delete IAM role |
| DRT role association | Disassociate via `disassociate-drt-role` |
| DRT log access | Disassociate via `disassociate-drt-log-bucket` |
| Proactive engagement | Disable via `disable-proactive-engagement` |

## Output Format

```yaml
root_cause: "srt_engagement — <specific_cause>"
evidence:
  - type: subscription
    content: "<Shield Advanced status>"
  - type: drt_access
    content: "<DRT role and log bucket access>"
  - type: proactive_engagement
    content: "<enabled/disabled, emergency contacts>"
severity: CRITICAL
mitigation:
  immediate: "Configure DRT access role and emergency contacts"
  long_term: "Enable proactive engagement with health checks on all protected resources"
```

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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
