---
title: "B2 — SigV4 Authentication Errors"
description: "Diagnose and resolve SigV4 signing failures for AMP remote write and query"
status: active
severity: HIGH
triggers:
  - "SigV4"
  - "sigv4"
  - "403 Forbidden"
  - "AccessDeniedException"
  - "authentication failed"
  - "signing"
  - "credentials"
owner: devops-agent
objective: "Identify and fix SigV4 authentication issues for AMP API access"
context: "AMP requires AWS SigV4 for all API requests. Both remote write and query endpoints reject requests without valid SigV4 signatures. Common issues include wrong IAM permissions, expired credentials, incorrect region in signing config, and misconfigured collector auth extensions."
---

## Phase 1 — Triage

MUST:
- Verify IAM permissions for the collector's role: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names aps:RemoteWrite aps:QueryMetrics aps:GetSeries aps:GetLabels`
- Check if credentials are valid: `aws sts get-caller-identity` (using the collector's credentials)
- Verify SigV4 region matches workspace region
- Check collector logs for specific auth error messages

SHOULD:
- Verify the IAM role trust policy allows the collector to assume it (EC2, EKS pod, Lambda)
- Check if credentials are being refreshed (not expired static keys)
- For EKS: verify IRSA or pod identity is configured correctly
- Test with curl: `curl -v --aws-sigv4 "aws:amz:<region>:aps" --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" "https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>/api/v1/query?query=up"`

MAY:
- Check CloudTrail for failed authentication events
- Verify no SCP or permission boundary blocks AMP actions
- Check if the signing service name is correct (`aps` for AMP)

## Phase 2 — Remediate

MUST:
- If missing IAM permissions: add required AMP actions to the role policy:
  ```json
  {
    "Effect": "Allow",
    "Action": [
      "aps:RemoteWrite",
      "aps:QueryMetrics",
      "aps:GetSeries",
      "aps:GetLabels",
      "aps:GetMetricMetadata"
    ],
    "Resource": "arn:aws:aps:<region>:<account>:workspace/<workspace-id>"
  }
  ```
- If expired credentials: rotate access keys or fix credential refresh mechanism
- If wrong region: update SigV4 config to match workspace region
- If wrong service name: use `aps` as the SigV4 service name

SHOULD:
- Use IAM roles (not static access keys) for collectors
- For EKS collectors: use IRSA or EKS Pod Identity
- For EC2 collectors: use instance profile
- Scope IAM permissions to specific workspace ARN

MAY:
- Enable CloudTrail logging for AMP API calls to detect auth failures
- Implement credential rotation automation
- Use AWS Secrets Manager for credential management

## Common Issues

- symptoms: "403 Forbidden with 'The security token included in the request is invalid'"
  diagnosis: "Expired or invalid AWS credentials."
  resolution: "Refresh credentials. If using static keys, rotate them. If using IRSA, check service account annotation and OIDC provider."

- symptoms: "403 Forbidden with 'User: arn:aws:iam::... is not authorized to perform: aps:RemoteWrite'"
  diagnosis: "IAM policy missing aps:RemoteWrite permission."
  resolution: "Add aps:RemoteWrite to the IAM policy attached to the collector's role."

- symptoms: "SignatureDoesNotMatch error"
  diagnosis: "SigV4 signing region or service name mismatch."
  resolution: "Verify sigv4 region matches workspace region. Service name must be 'aps'. Check system clock is synchronized (NTP)."

- symptoms: "ADOT collector logs show 'failed to get credentials'"
  diagnosis: "ADOT sigv4auth extension cannot obtain AWS credentials."
  resolution: "Verify ADOT has access to credentials: IRSA annotation, instance profile, or environment variables."

## Safety Ratings

```
safety_ratings:
  - "simulate-principal-policy: GREEN — read-only IAM check"
  - "sts get-caller-identity: GREEN — read-only credential check"
  - "add AMP IAM permissions: YELLOW — changes IAM authorization"
  - "rotate access keys: YELLOW — changes credentials, may break other services"
  - "configure IRSA: YELLOW — changes EKS service account trust"
```

## Escalation Conditions

- SigV4 authentication failures causing data loss in production monitoring
- Credential rotation needed across multiple collectors simultaneously
- IRSA configuration requires EKS cluster OIDC provider changes
- SCP or permission boundary blocks AMP actions at the organization level

## Data Sensitivity

- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws sts get-caller-identity`: LOW — reveals current principal identity
- IAM policies: MEDIUM — reveals authorization rules
- Collector credentials: HIGH — provide access to AMP workspace

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER use static access keys when IAM roles (IRSA, instance profiles) are available
- NEVER grant aps:* wildcard permissions — scope to specific actions and workspace ARN

## Phase 3 — Rollback

1. If IAM permissions were changed incorrectly, restore previous policy
2. If credentials were rotated and broke collectors, restore previous credentials if still valid
3. If IRSA configuration was changed, restore previous service account annotation and trust policy
4. Verify all collectors can authenticate successfully after rollback
5. Confirm metrics ingestion resumes by checking CloudWatch IngestionRate

## Output Format

```yaml
root_cause: "SigV4 authentication — <specific_cause>"
evidence:
  - type: iam_simulation
    content: "<IAM policy simulation results>"
  - type: collector_logs
    content: "<auth error messages>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or credential configuration"
  long_term: "Use IAM roles with least-privilege policies, implement credential rotation"
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
