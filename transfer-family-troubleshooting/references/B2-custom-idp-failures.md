---
title: "B2 — Custom Identity Provider Failures"
description: "Diagnose authentication failures when using Lambda or API Gateway custom identity providers"
status: active
severity: HIGH
triggers:
  - "custom IdP"
  - "Lambda authentication"
  - "API Gateway auth"
  - "identity provider error"
  - "custom authorizer"
  - "password auth failed"
owner: devops-agent
objective: "Identify why custom identity provider authentication is failing"
context: "Custom identity providers delegate authentication to a Lambda function or API Gateway. The provider must return a specific JSON response format with Role and HomeDirectory. Failures can occur in the Lambda execution, response format, IAM role configuration, or network connectivity to the IdP backend."
---

## Phase 1 — Triage

MUST:
- Verify server uses custom IdP: `aws transfer describe-server --server-id <server-id> --query 'Server.{IdentityProviderType:IdentityProviderType,IdentityProviderDetails:IdentityProviderDetails}'`
- Test the identity provider: `aws transfer test-identity-provider --server-id <server-id> --user-name <username> --user-password "<password>"`
- Check Lambda function logs (if Lambda IdP): `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Check Lambda function configuration: `aws lambda get-function --function-name <function-name>`
- Verify the Lambda execution role has required permissions

SHOULD:
- Validate the IdP response format contains Role and HomeDirectory at minimum
- Check Lambda timeout (must complete within 30 seconds for Transfer Family)
- Check if the Lambda function can reach external auth backends (LDAP, database) if in a VPC
- Review API Gateway execution logs if using API Gateway IdP

MAY:
- Test the Lambda function directly with a test event matching Transfer Family input format
- Check API Gateway stage configuration and deployment status
- Review Lambda concurrency limits and throttling

## Phase 2 — Remediate

MUST:
- Ensure Lambda returns the correct JSON format: `{"Role": "arn:aws:iam::...", "HomeDirectory": "/bucket/prefix", ...}`
- Fix Lambda errors (timeouts, exceptions, permission issues)
- Ensure the IAM role ARN returned by the IdP is valid and trusts transfer.amazonaws.com

SHOULD:
- Add error handling and logging to the Lambda function for debugging
- Set Lambda timeout to at least 10 seconds to handle backend latency
- Cache authentication results in the Lambda if querying external systems

MAY:
- Implement Lambda provisioned concurrency to avoid cold start delays
- Add CloudWatch alarms on Lambda errors and duration

## Common Issues

- symptoms: "test-identity-provider returns empty response"
  diagnosis: "Lambda function returns null, throws an unhandled exception, or times out."
  resolution: "Check Lambda logs for errors. Ensure the function returns a valid JSON response."

- symptoms: "Authentication succeeds in test but fails from SFTP client"
  diagnosis: "Lambda returns PublicKeys but the client's private key doesn't match any returned public key."
  resolution: "Verify PublicKeys in the Lambda response match the client's key pair."

- symptoms: "Lambda returns valid response but user gets access denied on file operations"
  diagnosis: "The Role ARN returned by Lambda does not trust transfer.amazonaws.com or lacks S3/EFS permissions."
  resolution: "Update the IAM role trust policy and attach appropriate S3 or EFS permissions."

## Output Format

```yaml
root_cause: "custom_idp_failure — <specific_cause>"
evidence:
  - type: idp_test
    content: "<test-identity-provider response>"
  - type: lambda_logs
    content: "<Lambda execution logs>"
  - type: idp_response
    content: "<response format validation>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda function or IdP response format"
  long_term: "Add monitoring, error handling, and caching to the IdP Lambda"
```


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

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
