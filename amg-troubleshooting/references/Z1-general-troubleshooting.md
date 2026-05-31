---
title: "Z1 — General Amazon Managed Grafana Troubleshooting (Catch-All)"
description: "Fallback SOP for AMG issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown AMG issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all workspaces: `aws grafana list-workspaces`
- Get workspace details: `aws grafana describe-workspace --workspace-id <workspace-id>`
- Check workspace status and health: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.{Status:status,Health:health}'`
- Check authentication configuration: `aws grafana describe-workspace-authentication --workspace-id <workspace-id>`
- List workspace permissions: `aws grafana list-permissions --workspace-id <workspace-id>`
- Check workspace IAM role: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.workspaceRoleArn'`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=grafana.amazonaws.com --max-results 20`
- Verify data source types enabled: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.dataSources'`
- Check VPC configuration: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.vpcConfiguration'`
- Test workspace endpoint accessibility: `curl -sI https://<workspace-endpoint>/api/health`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Workspace creation/update/config issues → Workspace (A1-A3)
- SSO/SAML/login failures → Authentication (B1-B3)
- Data source connection/query errors → Data Sources (C1-C4)
- Dashboard rendering/panel errors → Dashboards (D1-D3)
- Alert rule/notification failures → Alerting (E1-E2)
- Plugin missing/broken → Plugins (F1-F2)
- VPC/network connectivity → Networking (G1-G2)
- API key/IaC provisioning → API & Automation (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all workspace configurations systematically
- Review CloudTrail for recent configuration changes
- Verify IAM role permissions match workspace requirements
- Test data source connectivity via Grafana API

SHOULD:
- Check AWS Health Dashboard for AMG service events
- Compare with a known-good workspace configuration
- Review workspace CloudWatch metrics for anomalies

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

```
safety_ratings:
  - "list-workspaces: GREEN — read-only enumeration"
  - "describe-workspace: GREEN — read-only status check"
  - "describe-workspace-authentication: GREEN — read-only auth check"
  - "list-permissions: GREEN — read-only permission check"
  - "test workspace endpoint: GREEN — read-only health check"
```

## Escalation Conditions

- Workspace completely inaccessible and cannot be diagnosed via API
- Issue cannot be classified into any specific failure domain after systematic investigation
- Multiple failure domains affected simultaneously
- AWS Health Dashboard shows AMG service event in the workspace region

## Data Sensitivity

- `aws grafana describe-workspace`: LOW — workspace metadata
- `aws grafana list-permissions`: MEDIUM — reveals user access levels
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history
- Workspace health check: LOW — returns basic health status

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER make configuration changes during general troubleshooting without first identifying root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to pre-investigation state
2. Restore workspace settings if they were modified during troubleshooting
3. Verify all data sources, dashboards, and alerts function correctly
4. Document the investigation path and findings for the team
5. If the issue remains unresolved, escalate to AWS Support with collected evidence

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<workspace|authentication|data_sources|dashboards|alerting|plugins|networking|api_automation|unknown>"
investigation_path: "workspace status → authentication → data sources → <domain_classification>"
evidence:
  - type: workspace_config
    content: "<workspace configuration summary>"
  - type: authentication
    content: "<authentication configuration>"
  - type: cloudtrail
    content: "<relevant events>"
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
