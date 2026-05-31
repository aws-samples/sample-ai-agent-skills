---
title: "Z1 — General Global Accelerator Troubleshooting (Catch-All)"
description: "Fallback SOP for Global Accelerator issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Global Accelerator issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get accelerator overview: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn>`
- Check accelerator status and enabled state: `aws globalaccelerator describe-accelerator --accelerator-arn <accelerator-arn> --query '{Status:Accelerator.Status,Enabled:Accelerator.Enabled}'`
- List listeners: `aws globalaccelerator list-listeners --accelerator-arn <accelerator-arn>`
- Check endpoint groups and health: `aws globalaccelerator list-endpoint-groups --listener-arn <listener-arn>`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name NewFlowCount --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Test basic connectivity: `nc -zv <static-ip> <port>`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=globalaccelerator.amazonaws.com --max-results 10`
- Check accelerator attributes (flow logs): `aws globalaccelerator describe-accelerator-attributes --accelerator-arn <accelerator-arn>`
- Verify DNS resolution: `nslookup <accelerator-dns-name>`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Accelerator creation/config issues → Accelerator (A1-A3)
- Endpoint health/group/traffic issues → Endpoints (B1-B3)
- Client affinity / flow logs → Routing (C1-C2)
- Latency / throughput → Performance (D1-D2)
- DNS / custom routing → DNS (E1-E2)
- DDoS / cross-account → Security (F1-F2)
- ALB/NLB/EC2/EIP / custom routing accelerator → Integration (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all listener and endpoint group configurations systematically
- Review CloudTrail for recent configuration changes
- Check flow logs if enabled
- Verify endpoint health across all endpoint groups

SHOULD:
- Check AWS Health Dashboard for Global Accelerator service events
- Compare with a known-good accelerator configuration
- Test from multiple geographic locations

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<accelerator|endpoints|routing|performance|dns|security|integration|unknown>"
investigation_path: "accelerator config → listeners → endpoint groups → CloudWatch → <domain_classification>"
evidence:
  - type: accelerator_config
    content: "<accelerator configuration summary>"
  - type: cloudwatch
    content: "<flow counts and metrics>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
