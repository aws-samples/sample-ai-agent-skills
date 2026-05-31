# AWS KMS Diagnostics Skill

Agent skill for investigating and troubleshooting AWS KMS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS KMS when the console alone isn't enough — key policy analysis, grant management, cross-account access debugging, key state troubleshooting, encryption context issues, service integration problems, throttling analysis, and security auditing.

### Activate When

- Access denied on KMS operations
- Key policy evaluation confusion
- Default key policy issues
- Grant management problems
- Grant token errors
- Cross-account key access failures
- ViaService condition errors
- Disabled key errors
- Pending deletion recovery
- Key rotation configuration issues
- Encryption context mismatches
- Asymmetric key operation failures
- Multi-region key replication issues
- S3 SSE-KMS errors
- EBS encryption failures
- RDS encryption issues
- Request rate limit throttling (ThrottlingException)
- Key quota limits
- Key policy vs IAM policy conflicts
- CloudTrail audit logging gaps

---

## Skill Structure

```
kms-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-access-denied.md
    ├── A2-policy-evaluation.md
    ├── A3-default-key-policy.md
    ├── B1-grant-management.md
    ├── B2-grant-tokens.md
    ├── C1-cross-account-access.md
    ├── C2-viaservice-conditions.md
    ├── D1-disabled-key.md
    ├── D2-pending-deletion.md
    ├── D3-key-rotation.md
    ├── E1-encryption-context.md
    ├── E2-asymmetric-keys.md
    ├── E3-multi-region-keys.md
    ├── F1-s3-sse-kms.md
    ├── F2-ebs-encryption.md
    ├── F3-rds-encryption.md
    ├── G1-request-rate-limits.md
    ├── G2-key-quotas.md
    ├── H1-key-policy-vs-iam.md
    ├── H2-audit-logging.md
    ├── Z1-general-troubleshooting.md
    ├── kms-guardrails.md
    └── kms-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Key Policy** | A1–A3 | Access denied, policy evaluation, default key policy |
| **B — Grants** | B1–B2 | Grant management, grant tokens |
| **C — Cross-Account** | C1–C2 | Cross-account key access, ViaService conditions |
| **D — Key State** | D1–D3 | Disabled key, pending deletion, key rotation |
| **E — Encryption** | E1–E3 | Encryption context, asymmetric keys, multi-region keys |
| **F — Integration** | F1–F3 | S3 SSE-KMS, EBS encryption, RDS encryption |
| **G — Quotas** | G1–G2 | Request rate limits, key quotas |
| **H — Security** | H1–H2 | Key policy vs IAM policy, audit logging |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/kms-guardrails.md` covering key policy as primary access control, default key policy root statement, key deletion permanence, disabled key behavior, encryption context binding, grant temporary access, AWS-managed key restrictions, multi-region key architecture, request rate limits, key material origin, custom key store constraints, and key spec immutability.

---

## Investigation Workflow

1. **Triage** — Describe key metadata, get key policy, list grants, check key rotation status, review CloudTrail events
2. **Deep Dive** — Simulate IAM policies, check throttling metrics, examine multi-region configuration
3. **Detailed** — Inspect retirable grants, custom key stores, specific CloudTrail decrypt/encrypt events, service quotas

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `kms:Describe*`, `kms:Get*`, `kms:List*`, `iam:SimulatePrincipalPolicy`, `iam:GetRole`, `iam:GetPolicy`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`, `s3:GetBucketEncryption`, `ec2:DescribeVolumes`, `rds:DescribeDBInstances`, `service-quotas:GetServiceQuota`
- CloudTrail logging enabled (required for KMS audit)

---

## Usage Examples

```
# Describe a KMS key
aws kms describe-key --key-id <key-id>

# Get the key policy
aws kms get-key-policy --key-id <key-id> --policy-name default

# List grants on a key
aws kms list-grants --key-id <key-id>

# Check key rotation status
aws kms get-key-rotation-status --key-id <key-id>

# Check for KMS throttling in CloudTrail
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20

# Simulate IAM policy for KMS access
aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names kms:Decrypt --resource-arns <key-arn>
```

---

## License

MIT-0
