---
title: "Z1 — General Polly Troubleshooting"
description: "Catch-all runbook for unclassified Polly issues"
status: active
severity: MEDIUM
triggers: ["Polly not working", "something wrong with Polly"]
owner: devops-agent
objective: "Systematically triage unclassified Polly issues"
---

## Phase 1 — Triage

MUST:
- List voices: `aws polly describe-voices`
- List lexicons: `aws polly list-lexicons`
- Check recent tasks: `aws polly list-speech-synthesis-tasks`
- Test simple synthesis: `aws polly synthesize-speech --text "test" --output-format mp3 --voice-id Joanna output.mp3`

SHOULD:
- Check IAM permissions
- Check CloudTrail for errors
- Verify region

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B2, C1-C2, D1-D2, or E1-E2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — general troubleshooting uses read-only operations
- Service Disruption Risk: LOW — diagnostic operations do not affect synthesis
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Polly service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows Polly service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Service configurations — INTERNAL
- CloudWatch metrics — INTERNAL
- Error messages — INTERNAL
- API usage data — INTERNAL
- Never expose synthesis input text during troubleshooting

## Prohibited Actions
- Do NOT make changes to production configurations without approval
- Do NOT share service details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside Polly service boundaries
- Do NOT ignore recurring issues
- Do NOT expose API credentials during troubleshooting

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore service to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify service health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
