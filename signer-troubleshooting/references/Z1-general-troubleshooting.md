---
title: "Z1 — General AWS Signer Troubleshooting"
description: "Catch-all runbook for Signer issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Signer error", "signing issue", "general Signer problem"]
owner: devops-agent
objective: "Provide systematic troubleshooting for unclassified Signer issues"
context: "This runbook covers general AWS Signer issues. Start here when the issue is unclear, then route to a specific runbook once the domain is identified."
---

## Phase 1 — Triage

MUST:
- List all signing profiles: `aws signer list-signing-profiles --query 'profiles[*].{Name:profileName,Platform:platformId,Status:status}'`
- List recent signing jobs: `aws signer list-signing-jobs --status Failed --max-results 10`
- Verify IAM caller identity: `aws sts get-caller-identity`

SHOULD:
- Check CloudTrail for recent Signer API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=signer.amazonaws.com --max-results 10`
- Verify the AWS region is correct for the signing resources

MAY:
- Check AWS Health Dashboard for Signer service issues
- Review recent infrastructure changes affecting signing workflows

## Phase 2 — Remediate

MUST:
- Identify the specific issue domain (profiles, jobs, validation, permissions, integration)
- Route to the appropriate specific runbook (A1-F1) once domain is identified
- Verify basic prerequisites: IAM permissions, S3 access, profile status

SHOULD:
- Check if the issue affects a single signing workflow or multiple
- Determine if the issue started after a recent change

MAY:
- Collect signing job details for detailed analysis
- Open an AWS Support case if the issue appears service-side

## Common Issues

- symptoms: "Signing operations fail with generic errors"
  diagnosis: "IAM permissions, S3 access, or profile status issue."
  resolution: "Check IAM permissions, S3 bucket policies, and signing profile status."

- symptoms: "Signing worked before but stopped working"
  diagnosis: "Profile revoked, signature expired, or IAM policy changed."
  resolution: "Check profile status, signature validity, and recent IAM changes."

## Safety Ratings

```
safety_ratings:
  - "list-signing-profiles: GREEN — read-only profile listing"
  - "list-signing-jobs: GREEN — read-only job listing"
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
```

## Escalation Conditions

- Multiple unrelated Signer features fail simultaneously
- Issue affects all signing operations in the account
- Signer service health shows active incidents
- Issue persists after all standard troubleshooting steps

## Data Sensitivity

- `aws signer list-signing-profiles`: LOW — profile metadata only
- `aws signer list-signing-jobs`: MEDIUM — reveals artifact locations
- `aws sts get-caller-identity`: LOW — current IAM identity
- `aws cloudtrail lookup-events`: MEDIUM — reveals API activity

## Prohibited Actions

- NEVER make broad changes without isolating the issue first
- NEVER revoke signing profiles as a troubleshooting step
- NEVER assume a service outage without checking AWS Health Dashboard
- NEVER skip triage and jump directly to remediation

## Phase 3 — Rollback

1. Document all changes made during troubleshooting
2. Revert changes in reverse order if they did not resolve the issue
3. Verify the environment is in the same state as before troubleshooting
4. Collect all diagnostic data for escalation if unresolved
5. Document troubleshooting steps and results

## Output Format

```yaml
root_cause: "general_signer — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook or apply general fix"
  long_term: "Implement signing workflow monitoring"
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
