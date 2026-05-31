---
title: "Z1 — General WorkSpaces Web Troubleshooting"
description: "Catch-all runbook for WorkSpaces Web issues"
status: active
severity: MEDIUM
triggers: ["WorkSpaces Web error", "Secure Browser not working", "portal issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified WorkSpaces Web issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List portals: `aws workspaces-web list-portals --query 'portals[*].{PortalArn:portalArn,Status:portalStatus,Endpoint:portalEndpoint}'`
- Check portal details: `aws workspaces-web get-portal --portal-arn <portal-arn>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workspaces-web.amazonaws.com --max-results 20`

SHOULD:
- Check associated resources (browser settings, network settings, IdP)
- Verify the portal endpoint URL is accessible from the user's network

MAY:
- Check AWS Health Dashboard for WorkSpaces Web service issues
- Review recent configuration changes across all portal resources

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)
- Document findings from initial triage

SHOULD:
- Classify the issue by category (portal, session, identity, network, policy, access)
- Collect relevant evidence before escalating

MAY:
- Set up temporary monitoring for the specific issue
- Contact AWS Support with collected evidence if no runbook matches

## Safety Ratings

```
safety_ratings:
  - "list-portals: GREEN — read-only enumeration"
  - "get-portal: GREEN — read-only portal details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Issue does not match any specific runbook after systematic investigation
- Multiple portals affected simultaneously suggesting service-level issue
- Issue requires AWS internal investigation
- Compliance or security incident related to WorkSpaces Web

## Data Sensitivity

- `aws workspaces-web list-portals`: LOW — returns portal metadata
- `aws workspaces-web get-portal`: LOW — returns portal configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER access user session data without authorization
- NEVER modify identity provider settings as part of general troubleshooting
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. Revert any configuration changes made during investigation
2. Restore portal settings if they were modified during troubleshooting
3. Verify all portals are in ACTIVE state before closing
4. Document all investigation steps and findings
5. Verify user access is functioning normally before closing the investigation

## Common Issues

- symptoms: "Portal was working yesterday but not today"
  diagnosis: "Check for IdP certificate expiry, network changes, or IP access rule changes."
  resolution: "Follow triage steps to identify the specific domain, then pivot to the appropriate runbook."

## Output Format

```yaml
root_cause: "general_workspaces_web — <specific_cause>"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
