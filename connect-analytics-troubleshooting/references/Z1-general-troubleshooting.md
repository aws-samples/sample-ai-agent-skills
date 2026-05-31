---
title: "Z1 — General Contact Lens & Wisdom Troubleshooting"
description: "Catch-all runbook for Contact Lens and Wisdom issues"
status: active
severity: MEDIUM
triggers: ["Contact Lens error", "Wisdom error", "analytics not working", "Connect analytics issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Contact Lens or Wisdom issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws connect describe-instance --instance-id <instance-id>`
- List recent contacts: `aws connect list-contacts --instance-id <instance-id> --time-range '{"Type":"INITIATION_TIMESTAMP","StartTime":"<start>","EndTime":"<end>"}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=connect.amazonaws.com --max-results 20`

SHOULD:
- Check Contact Lens enablement status
- Verify Wisdom knowledge bases and assistants
- Check S3 bucket for analytics output

MAY:
- Check AWS Health Dashboard for Connect service issues
- Review recent configuration changes

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-G1)
- Document findings from initial triage

SHOULD:
- Classify the issue by category (enablement, analysis, rules, Wisdom, quality, export, integration)
- Collect relevant evidence before escalating

MAY:
- Set up temporary monitoring for the specific issue
- Contact AWS Support with collected evidence if no runbook matches

## Safety Ratings

```
safety_ratings:
  - "describe-instance: GREEN — read-only instance details"
  - "list-contacts: GREEN — read-only contact listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Issue does not match any specific runbook after systematic investigation
- Multiple features affected simultaneously suggesting service-level issue
- Issue requires AWS internal investigation
- Compliance or regulatory issue related to Contact Lens data

## Data Sensitivity

- `aws connect describe-instance`: LOW — returns instance metadata
- `aws connect list-contacts`: MEDIUM — returns contact metadata including agent and queue info
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER access call recordings or transcripts without authorization
- NEVER modify contact flows as part of general troubleshooting
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. Revert any configuration changes made during investigation
2. Restore instance settings if they were modified during troubleshooting
3. Verify Contact Lens and Wisdom are functioning normally
4. Document all investigation steps and findings
5. Verify analytics are being generated for new contacts before closing

## Common Issues

- symptoms: "Contact Lens was working yesterday but not today"
  diagnosis: "Check for contact flow changes, S3 bucket policy changes, or IAM permission changes."
  resolution: "Follow triage steps to identify the specific domain, then pivot to the appropriate runbook."

## Output Format

```yaml
root_cause: "general_connect_analytics — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and documentation"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
