---
title: "E1 — Cross-Account Event Delivery Issues"
description: "Diagnose cross-account event delivery failures"
status: active
severity: HIGH
triggers:
  - "cross-account events"
  - "events not delivered cross-account"
  - "PutEvents cross-account"
  - "event bus another account"
owner: devops-agent
objective: "Identify and resolve cross-account event delivery issues"
context: "Cross-account event delivery requires a resource policy on the destination event bus and IAM permissions in the source account. Both accounts must be in the same region. Issues include missing policies, incorrect ARNs, and permission mismatches."
---

## Phase 1 — Triage

MUST:
- Check destination event bus policy: `aws events describe-event-bus --name <bus-arn>`
- Verify source account IAM role has events:PutEvents permission for the destination bus ARN
- Check the destination bus ARN format: arn:aws:events:<region>:<account-id>:event-bus/<bus-name>
- Verify both accounts are in the same region

SHOULD:
- Check CloudTrail in both accounts for PutEvents calls
- Verify the event bus policy grants the source account events:PutEvents
- Check for SCP restrictions in either account
- Test with a simple event to isolate the issue

MAY:
- Check for organization-level event bus policies
- Verify no condition keys in the policy are blocking access
- Review cross-account networking if using VPC endpoints

## Phase 2 — Remediate

MUST:
- Add resource policy on destination bus granting source account events:PutEvents
- Add IAM policy in source account allowing events:PutEvents on destination bus ARN
- Verify both accounts are in the same region

SHOULD:
- Use specific account IDs in policies (not wildcards)
- Test cross-account delivery after policy changes
- Document cross-account event flow architecture

MAY:
- Use AWS Organizations for organization-wide event bus policies
- Implement cross-account event monitoring

## Common Issues

- symptoms: "PutEvents returns success but events not received in destination"
  diagnosis: "Destination event bus policy does not grant the source account access."
  resolution: "Add events:PutEvents permission for the source account in the destination bus policy."

- symptoms: "AccessDeniedException on PutEvents to cross-account bus"
  diagnosis: "Source account IAM role lacks permission to call PutEvents on the destination bus ARN."
  resolution: "Add events:PutEvents for the destination bus ARN to the source account's IAM policy."

- symptoms: "Cross-account delivery works for some events but not others"
  diagnosis: "Destination bus policy has condition keys filtering specific event patterns."
  resolution: "Review policy conditions. Ensure they match the events being sent."

## Output Format

```yaml
root_cause: "cross_account_delivery — <specific_cause>"
evidence:
  - type: bus_policy
    content: "<destination event bus resource policy>"
  - type: iam_policy
    content: "<source account IAM permissions>"
  - type: cloudtrail
    content: "<PutEvents call results in both accounts>"
severity: HIGH
mitigation:
  immediate: "Fix resource policy and IAM permissions"
  long_term: "Document cross-account architecture and implement monitoring"
```


## Safety Ratings
```
safety_ratings:
  - "Check destination event bus policy: GREEN — read-only API call"
  - "Verify source account IAM role: GREEN — read-only IAM inspection"
  - "Check CloudTrail in both accounts: GREEN — read-only audit log query"
  - "Add resource policy on destination bus: YELLOW — grants cross-account access"
  - "Add IAM policy in source account: YELLOW — grants cross-account event delivery permission"
  - "Use organization-wide bus policies: YELLOW — broad access grant across organization"
```

## Escalation Conditions
- Cross-account event delivery serves production integration
- Policy changes require coordination between account owners
- SCP restrictions blocking cross-account access
- Organization-level policy changes affecting multiple accounts
- Security review required for new cross-account access grants

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Account IDs: AWS account identifiers"
    - "Event bus policies: cross-account access configuration"
    - "IAM policies: permission grants across account boundaries"
    - "Event payloads: business data crossing account boundaries"
  handling: "Cross-account configurations expose account relationships. Use specific account IDs, never wildcards."
```

## Prohibited Actions
- NEVER use wildcard (*) principals in event bus resource policies for production
- NEVER grant cross-account access without explicit approval from both account owners
- NEVER expose event bus policies containing account IDs externally
- NEVER remove cross-account policies without confirming no active event flows depend on them

## Phase 3 — Rollback
- If destination bus policy was modified: restore previous resource policy
- If source account IAM policy was changed: revert to previous IAM policy version
- If organization-wide policy was applied: remove the organization condition from the bus policy
- If cross-account delivery was enabled: remove both the bus policy and source IAM policy to fully revoke access

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
