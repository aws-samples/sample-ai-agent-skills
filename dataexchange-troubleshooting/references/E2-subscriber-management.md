---
title: "E2 — Subscriber Management"
description: "Diagnose subscriber management and approval workflow issues"
status: active
severity: MEDIUM
triggers: ["subscriber approval failed", "subscription request pending", "subscriber access revoked", "subscriber notification error"]
owner: devops-agent
objective: "Resolve subscriber management issues including approval workflows, access control, and subscriber notifications"
context: "Providers manage subscriber access through approval workflows. Some products auto-approve, others require manual review. Providers can revoke access."
---

## Phase 1 — Triage

MUST:
- List data set subscribers: `aws dataexchange list-data-sets --query 'DataSets[*].{Name:Name,Id:Id,OriginDetails:OriginDetails}'`
- Check pending subscription requests (provider side)
- Review subscriber entitlement status

SHOULD:
- Verify approval workflow configuration
- Check notification settings for subscription events
- Review subscriber access patterns

MAY:
- Check CloudTrail for subscription approval/rejection events
- Review subscriber feedback or support requests

## Phase 2 — Remediate

MUST:
- Process pending subscription approvals or rejections
- Fix notification delivery for subscription events
- Ensure subscriber access matches entitlement status

SHOULD:
- Configure auto-approval for appropriate products
- Set up subscription event notifications via SNS

MAY:
- Implement subscriber onboarding documentation
- Set up usage monitoring for subscriber access patterns

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-data-sets: GREEN — read-only listing"
  - "subscription approval: YELLOW — grants data access to subscriber"
  - "subscription revocation: RED — removes subscriber data access"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Subscription approval workflow is broken preventing all new subscribers
- Subscriber access was revoked incorrectly and needs restoration
- Large number of pending approvals causing subscriber delays
- Subscriber reports unauthorized data access

## Data Sensitivity

- `subscriber list`: MEDIUM — reveals subscriber account IDs and access patterns
- `subscription requests`: MEDIUM — contains subscriber business information
- `approval workflow configuration`: LOW — contains workflow rules
- `subscriber usage data`: MEDIUM — reveals data consumption patterns

## Prohibited Actions

- NEVER approve subscriptions without verifying subscriber identity for restricted data
- NEVER revoke subscriber access without proper notification and grace period
- NEVER share subscriber information with other subscribers
- NEVER auto-approve subscriptions for sensitive data products without review

## Phase 3 — Rollback

1. If a subscription was incorrectly revoked, re-approve the subscriber
2. If auto-approval was enabled incorrectly, disable and review recent approvals
3. If notifications were misconfigured, restore previous settings
4. Verify affected subscribers have correct access
5. Document the management issue and update approval procedures

## Common Issues

- symptoms: "Subscribers not receiving approval notifications"
  diagnosis: "SNS notification topic not configured or subscriber email not verified."
  resolution: "Configure SNS topic for subscription events and verify subscriber endpoints."

## Output Format

```yaml
root_cause: "subscriber_management — <specific_cause>"
evidence:
  - type: subscriber_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix subscriber management issue"
  long_term: "Automate approval workflows and notifications"
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
