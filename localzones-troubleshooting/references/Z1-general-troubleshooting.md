---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Local Zones issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Local Zone error", "Local Zone not working", "general Local Zone issue", "unknown Local Zone problem"]
owner: devops-agent
objective: "Systematically diagnose and resolve general Local Zones issues"
context: "Use this runbook when the issue does not clearly match a specific runbook category. Follow the systematic investigation workflow to identify the root cause domain."
---

## Phase 1 — Triage

MUST:
- Check Local Zone opt-in status: `aws ec2 describe-availability-zones --all-availability-zones --filters "Name=zone-type,Values=local-zone"`
- List resources in Local Zones: `aws ec2 describe-instances --filters "Name=availability-zone,Values=<local-zone-name>"`
- Check CloudTrail for recent errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20`

SHOULD:
- Verify IAM permissions for EC2 operations
- Check AWS Health Dashboard for Local Zone service events
- Review recent infrastructure changes

MAY:
- Check VPC configuration and routing
- Review security group and NACL rules
- Test connectivity from Local Zone instances

## Phase 2 — Remediate

MUST:
- Identify the specific domain (enablement, instances, networking, storage, integration) and route to appropriate runbook
- Fix IAM permission issues if API calls are failing
- Verify Local Zone is opted in and available

SHOULD:
- Document the issue pattern for future reference
- Check if the issue affects multiple Local Zones or is isolated

MAY:
- Open AWS Support case if issue appears service-side
- Review recent changes that may have caused the issue

## Safety Ratings

```
safety_ratings:
  - "describe-availability-zones: GREEN — read-only zone check"
  - "describe-instances: GREEN — read-only instance enumeration"
  - "cloudtrail lookup-events: GREEN — read-only audit log"
  - "Any remediation action: varies — refer to specific runbook"
```

## Escalation Conditions

- Issue affects all resources in a Local Zone
- Problem persists after following specific runbook remediation
- Issue appears to be an AWS service-side problem
- Multiple unrelated Local Zone features failing simultaneously

## Data Sensitivity

- Zone status information: LOW — availability metadata
- Instance details: MEDIUM — reveals deployment configuration
- CloudTrail events: MEDIUM — reveals IAM principals and actions
- Network configuration: MEDIUM — reveals infrastructure topology

## Prohibited Actions

- NEVER terminate instances or delete resources as a troubleshooting step
- NEVER opt out of Local Zones with active resources
- NEVER modify VPC configuration without understanding impact
- NEVER assume a service-side issue without ruling out configuration problems

## Phase 3 — Rollback

1. If any resources were modified during troubleshooting, revert to pre-investigation state
2. Document all changes made during investigation
3. If wrong runbook was followed, undo all remediation steps
4. Verify all resources are in expected state after investigation

## Common Issues

- symptoms: "All EC2 API calls for Local Zone return errors"
  diagnosis: "IAM permissions missing or Local Zone not opted in."
  resolution: "Verify IAM policy includes EC2 permissions. Check Local Zone opt-in status."

- symptoms: "Resources work in parent region but not in Local Zone"
  diagnosis: "Service or feature not available in Local Zone."
  resolution: "Check service availability in the specific Local Zone. Use parent region for unsupported services."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<triage findings and domain identification>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive Local Zone monitoring"
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
