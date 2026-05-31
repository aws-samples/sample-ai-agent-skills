---
title: "F1 — Global Accelerator DDoS Protection Issues"
description: "Diagnose DDoS protection and Shield integration issues"
status: active
severity: HIGH
triggers:
  - "DDoS attack"
  - "Shield protection"
  - "DDoS protection"
  - "attack mitigation"
owner: devops-agent
objective: "Verify and troubleshoot DDoS protection for Global Accelerator"
context: "Shield Standard is automatically included with Global Accelerator. Shield Advanced provides enhanced protection, DDoS response team access, and cost protection. Shield Advanced requires a separate subscription."
---

## Phase 1 — Triage

MUST:
- Check if Shield Advanced is associated: `aws shield describe-protection --resource-arn <accelerator-arn>`
- List Shield protections: `aws shield list-protections --query "Protections[?ResourceArn=='<accelerator-arn>']"`
- Check for active DDoS events: `aws shield list-attacks --start-time <start> --end-time <end> --resource-arns <accelerator-arn>`
- Monitor traffic metrics for anomalies: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name NewFlowCount --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Check Shield Advanced subscription status: `aws shield get-subscription-state`
- Review attack details if detected: `aws shield describe-attack --attack-id <attack-id>`
- Check processed bytes for traffic spikes: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name ProcessedBytesIn --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 60 --statistics Sum`

MAY:
- Check if DDoS response team (DRT) has been engaged
- Review Shield Advanced cost protection eligibility
- Check AWS Health Dashboard for DDoS-related notifications

## Phase 2 — Remediate

MUST:
- Verify Shield Standard is active (automatic, no action needed)
- If under active attack, engage AWS Support (Shield Advanced customers get DRT access)

SHOULD:
- Enable Shield Advanced for enhanced protection: `aws shield create-protection --name <name> --resource-arn <accelerator-arn>`
- Configure Shield Advanced proactive engagement if available

MAY:
- Set up CloudWatch alarms for traffic anomalies
- Implement rate limiting at the application level

## Common Issues

- symptoms: "Suspected DDoS but no Shield events"
  diagnosis: "Shield Standard mitigates common attacks automatically without generating events."
  resolution: "Shield Standard handles L3/L4 attacks silently. For visibility, enable Shield Advanced."

- symptoms: "Shield Advanced protection not showing for accelerator"
  diagnosis: "Shield Advanced protection not created for this specific resource."
  resolution: "Create Shield Advanced protection for the accelerator ARN."

- symptoms: "High traffic but not a DDoS attack"
  diagnosis: "Legitimate traffic spike, not an attack."
  resolution: "Scale endpoints to handle increased traffic. Monitor for actual attack patterns."

## Output Format

```yaml
root_cause: "ddos_protection — <specific_cause>"
evidence:
  - type: shield_protection
    content: "<Shield protection status>"
  - type: attack_events
    content: "<active or recent attacks>"
  - type: traffic_metrics
    content: "<traffic anomaly data>"
severity: HIGH
mitigation:
  immediate: "Engage DRT if under active attack with Shield Advanced"
  long_term: "Enable Shield Advanced and configure proactive engagement"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
