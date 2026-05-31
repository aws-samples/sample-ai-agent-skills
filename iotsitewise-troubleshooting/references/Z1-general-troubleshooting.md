---
title: "Z1 — General SiteWise Troubleshooting"
description: "Catch-all runbook for unclassified SiteWise issues"
status: active
severity: MEDIUM
triggers:
  - "SiteWise not working"
  - "something wrong with SiteWise"
  - "SiteWise error"
owner: devops-agent
objective: "Systematically triage unclassified SiteWise issues"
context: "Use this runbook when the issue doesn't clearly match a specific category."
---

## Phase 1 — Triage

MUST:
- List asset models: `aws iotsitewise list-asset-models`
- Check gateway status: `aws iotsitewise list-gateways`
- Check portal status: `aws iotsitewise list-portals`
- Review CloudTrail for recent errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iotsitewise.amazonaws.com --max-results 20`

SHOULD:
- Check CloudWatch SiteWise metrics for errors
- Verify IAM permissions for the calling identity
- Check service health in the region

MAY:
- Review recent API calls for patterns
- Check Service Quotas usage

## Phase 2 — Route to specific runbook

Based on triage findings, route to:
- A1-A2 for asset model issues
- B1-B3 for property issues
- C1-C2 for portal issues
- D1-D2 for gateway issues
- E1-E2 for ingestion issues
- F1-F2 for alarm/notification issues

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<initial findings>"
severity: MEDIUM
mitigation:
  immediate: "Follow specific runbook"
  long_term: "Implement comprehensive monitoring"
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
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
