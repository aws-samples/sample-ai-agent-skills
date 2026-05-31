---
title: "C2 — IAM Authentication"
description: "Diagnose Neptune IAM authentication and SigV4 signing issues"
status: active
severity: HIGH
triggers:
  - "iam auth failed"
  - "sigv4 error"
  - "signature mismatch"
  - "access denied neptune"
owner: devops-agent
objective: "Identify and resolve Neptune IAM authentication issues"
context: "Neptune IAM auth requires AWS Signature Version 4 signing on all HTTP requests. There is no username/password authentication. SigV4 signing must include the correct region, service name, and credentials."
---

## Phase 1 — Triage

MUST:
- Verify IAM auth is enabled: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].IAMDatabaseAuthenticationEnabled'`
- Check caller identity: `aws sts get-caller-identity`
- Verify IAM policy allows neptune-db:connect: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names neptune-db:connect --resource-arns arn:aws:neptune-db:<region>:<account>:<cluster-resource-id>/*`
- Test unsigned request to confirm IAM auth is the issue: `curl -s https://<endpoint>:8182/status`

SHOULD:
- Check IAM policy for neptune-db:* permissions
- Verify SigV4 signing is using correct region and service name (neptune-db)
- Check for clock skew between client and AWS (SigV4 requires synchronized clocks)
- Verify the resource ID in the IAM policy matches the cluster resource ID

MAY:
- Enable CloudTrail logging for neptune-db API calls
- Test with AWS CLI using --debug flag for signing details
- Check for VPC endpoint policy restrictions

## Phase 2 — Remediate

MUST:
- Fix IAM policy to grant neptune-db:connect on the correct resource ARN
- Ensure SigV4 signing uses correct region and service name
- Synchronize client clock if clock skew detected

SHOULD:
- Use neptune-python-utils or AWS SDK for automatic SigV4 signing
- Configure IAM roles instead of access keys where possible
- Use resource-level permissions for least privilege

MAY:
- Implement credential rotation
- Set up CloudTrail monitoring for auth failures

## Common Issues

- symptoms: "403 Forbidden on all requests"
  diagnosis: "IAM policy does not grant neptune-db:connect permission."
  resolution: "Add neptune-db:connect permission for the cluster resource ID."

- symptoms: "Signature expired error"
  diagnosis: "Client clock is skewed more than 5 minutes from AWS time."
  resolution: "Synchronize client clock with NTP. SigV4 requires clock accuracy."

- symptoms: "Works with CLI but not application"
  diagnosis: "Application is not signing requests with SigV4 correctly."
  resolution: "Use AWS SDK or neptune-python-utils for automatic SigV4 signing."

## Output Format

```yaml
root_cause: "iam_auth — <specific_cause>"
evidence:
  - type: iam_config
    content: "<IAM auth enabled status>"
  - type: iam_policy
    content: "<IAM policy evaluation>"
  - type: signing
    content: "<SigV4 signing details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or SigV4 signing"
  long_term: "Use SDK-based signing and IAM roles"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — IAM authentication controls database access
- Service Disruption Risk: MEDIUM — auth failures prevent legitimate access
- Compliance Impact: HIGH — authentication directly affects security compliance

## Escalation Conditions
- IAM authentication completely blocks all database access
- SigV4 signing errors persist after credential refresh
- IAM policy changes cause widespread authentication failures
- Temporary credentials expire faster than expected
- Cross-account IAM authentication fails

## Data Sensitivity
- IAM policies and role ARNs — CONFIDENTIAL
- SigV4 signing credentials — RESTRICTED
- Authentication error logs — INTERNAL
- Cross-account trust configurations — CONFIDENTIAL
- Never expose IAM credentials or session tokens

## Prohibited Actions
- Do NOT disable IAM authentication on production clusters
- Do NOT use long-term IAM credentials for Neptune authentication
- Do NOT grant overly broad Neptune IAM permissions (neptune-db:*)
- Do NOT share IAM credentials across applications
- Do NOT bypass IAM authentication for troubleshooting
- Do NOT log SigV4 signatures or session tokens

## Phase 3 — Rollback
- Restore previous IAM policies if changes caused authentication failures
- Rotate credentials if they may have been exposed during troubleshooting
- Revert cluster IAM authentication settings if modified
- Re-establish cross-account trust relationships if broken
- Verify all applications can authenticate after changes
- Document IAM changes and their impact on authentication

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
