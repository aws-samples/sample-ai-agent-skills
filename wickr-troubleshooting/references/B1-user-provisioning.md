---
title: "B1 — User Provisioning"
description: "Diagnose user provisioning failures"
status: active
severity: MEDIUM
triggers: ["user not provisioned", "cannot add user", "provisioning failed", "SCIM error"]
owner: devops-agent
objective: "Resolve user provisioning and directory sync issues"
context: "Users can be provisioned via admin console, SCIM, or directory sync. Each method has different requirements."
---

## Phase 1 — Triage

MUST:
- Verify network is active: `aws wickr get-network --network-id <id>`
- Check CloudTrail for provisioning events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`
- Verify IAM permissions for user management operations

SHOULD:
- Check SCIM endpoint configuration if using automated provisioning
- Verify directory sync status and last sync time
- Review provisioning error logs: `aws logs filter-log-events --log-group-name /aws/wickr/<network-id> --filter-pattern "provisioning"`

MAY:
- Test provisioning a single user manually via admin console
- Check identity provider SCIM configuration

## Phase 2 — Remediate

MUST:
- Fix SCIM endpoint URL and authentication token if SCIM provisioning fails
- Ensure user email addresses are unique within the network
- Verify the network has capacity for additional users

SHOULD:
- Sync directory changes manually if automatic sync is delayed
- Verify user attributes match required schema (email, display name)

MAY:
- Implement automated provisioning via SCIM for large user bases
- Set up provisioning error notifications

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "user provisioning: YELLOW — adds users to the network"
  - "SCIM configuration: YELLOW — modifies provisioning pipeline"
```

## Escalation Conditions

- Bulk provisioning fails for all users simultaneously
- SCIM endpoint returns persistent authentication errors
- Users provisioned but cannot log into Wickr client
- Directory sync removes users unexpectedly

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- User provisioning data: HIGH — contains user email addresses and identity information
- SCIM tokens: HIGH — authentication credentials for provisioning
- Directory sync logs: MEDIUM — reveals user identity information

## Prohibited Actions

- NEVER share SCIM authentication tokens in logs or outputs
- NEVER bulk-delete users without explicit approval
- NEVER provision users to the wrong Wickr network
- NEVER bypass identity verification for user provisioning

## Phase 3 — Rollback

1. If users were provisioned to the wrong network, remove them from the incorrect network
2. If SCIM configuration was changed, restore original endpoint and token
3. If directory sync was modified, restore original sync configuration
4. Verify legitimate users retain access after changes
5. Document provisioning changes and update user inventory

## Common Issues

- symptoms: "SCIM provisioning returns 401 Unauthorized"
  diagnosis: "SCIM authentication token is expired or incorrect."
  resolution: "Generate a new SCIM token in the Wickr admin console and update the identity provider."

- symptoms: "User provisioned but cannot find Wickr network in client"
  diagnosis: "User email in Wickr does not match their login email."
  resolution: "Verify user email consistency between identity provider and Wickr provisioning."

## Output Format

```yaml
root_cause: "user_provisioning — <specific_cause>"
evidence:
  - type: provisioning_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix provisioning issue"
  long_term: "Automate provisioning with SCIM and monitoring"
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
