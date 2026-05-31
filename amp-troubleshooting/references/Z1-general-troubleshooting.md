---
title: "Z1 — General AMP Troubleshooting (Catch-All)"
description: "Fallback SOP for AMP issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown AMP issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check workspace status: `aws amp describe-workspace --workspace-id <workspace-id>`
- Check CloudWatch AMP metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/Prometheus \
    --metric-name ActiveSeries --dimensions Name=WorkspaceId,Value=<workspace-id> \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 --statistics Maximum
  ```
- Check ingestion errors: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name IngestionErrors --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Sum`
- Check query errors: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name QueryErrors --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Sum`
- Check CloudTrail for recent AMP API events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=aps.amazonaws.com --max-results 20`

SHOULD:
- Check alert manager definition: `aws amp describe-alert-manager-definition --workspace-id <workspace-id>`
- Check ruler namespaces: `aws amp list-rule-groups-namespaces --workspace-id <workspace-id>`
- Test remote write endpoint connectivity
- Test query endpoint with a simple query like `up`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Workspace not ACTIVE → Workspace (A1-A3)
- Ingestion errors or 403/429 on remote write → Ingestion (B1-B4)
- Query timeouts or errors → Query (C1-C3)
- Rules not evaluating or alerts not firing → Alerting (D1-D3)
- Collector pods failing or not sending → Collectors (E1-E3)
- IAM/auth errors → Security (F1-F2)
- Grafana or tool connectivity issues → Integration (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all workspace configurations systematically
- Review CloudTrail for recent configuration changes
- Check collector logs for any errors
- Verify IAM permissions for all principals accessing the workspace

SHOULD:
- Check AWS Health Dashboard for AMP service events
- Compare with a known-good workspace configuration
- Check VPC endpoint status if using private connectivity

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

```
safety_ratings:
  - "describe-workspace: GREEN — read-only status check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric check"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "describe-alert-manager-definition: GREEN — read-only config check"
  - "test remote write/query endpoints: GREEN — connectivity test"
```

## Escalation Conditions

- Workspace completely non-functional and cannot be diagnosed
- Issue cannot be classified into any specific failure domain
- Multiple failure domains affected simultaneously
- AWS Health Dashboard shows AMP service event in the workspace region

## Data Sensitivity

- `aws amp describe-workspace`: LOW — workspace metadata
- CloudWatch AMP metrics: LOW — aggregated operational metrics
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Alert manager and ruler configurations: MEDIUM — reveal alerting logic

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER make configuration changes during general troubleshooting without first identifying root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore alert manager and ruler configurations if they were modified
3. Verify all collectors are sending data and queries are functioning
4. Document the investigation path and findings for the team
5. If the issue remains unresolved, escalate to AWS Support with collected evidence

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<workspace|ingestion|query|alerting|collectors|security|integration|unknown>"
investigation_path: "workspace status → CloudWatch metrics → CloudTrail → <domain_classification>"
evidence:
  - type: workspace_status
    content: "<workspace configuration summary>"
  - type: cloudwatch_metrics
    content: "<AMP metrics summary>"
  - type: cloudtrail
    content: "<relevant API events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
