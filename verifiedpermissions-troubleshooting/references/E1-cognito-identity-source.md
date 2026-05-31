---
title: "E1 — Cognito Identity Source"
description: "Diagnose Cognito user pool identity source integration"
status: active
severity: HIGH
triggers:
  - "Cognito identity source error"
  - "user pool integration"
  - "Cognito token mapping"
owner: devops-agent
objective: "Resolve Cognito identity source configuration issues"
context: "Cognito identity sources map user pool tokens to Cedar entities. User pool groups can map to Cedar parent entities. Token claims become entity attributes. The identity source must reference a valid Cognito user pool."
---

## Phase 1 — Triage

MUST:
- List identity sources: `aws verifiedpermissions list-identity-sources --policy-store-id <id>`
- Get identity source details: `aws verifiedpermissions get-identity-source --policy-store-id <id> --identity-source-id <source-id>`
- Verify Cognito user pool exists: `aws cognito-idp describe-user-pool --user-pool-id <pool-id>`
- Check user pool app client: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>`

SHOULD:
- Verify token claim mapping configuration
- Check Cognito group-to-entity mapping
- Verify user pool is in the same region as the policy store

## Phase 2 — Remediate

MUST:
- Create identity source: `aws verifiedpermissions create-identity-source --policy-store-id <id> --configuration '{"cognitoUserPoolConfiguration":{"userPoolArn":"arn:aws:cognito-idp:<region>:<account>:userpool/<pool-id>"}}'  --principal-entity-type "MyApp::User"`
- Verify mapping works with is-authorized-with-token

SHOULD:
- Configure group entity type for Cognito groups
- Test with tokens from different user pool groups

## Common Issues

- symptoms: "User pool ARN not valid"
  diagnosis: "User pool doesn't exist or is in a different region."
  resolution: "Verify user pool ARN and ensure it's in the same region as the policy store."

- symptoms: "Groups not mapped to Cedar entities"
  diagnosis: "Group entity type not configured in identity source."
  resolution: "Update identity source with groupConfiguration specifying the group entity type."


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Policy stores contain complete authorization logic
- Schema definitions reveal application data model
- Authorization decisions log access patterns
- Classification: CONFIDENTIAL - authorization policies and access decision data

## Prohibited Actions

- NEVER delete policy stores used by production applications
- NEVER modify Cedar policies without testing authorization decisions first
- NEVER change schema definitions that could invalidate existing policies
- NEVER remove identity sources connected to active authentication flows

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
