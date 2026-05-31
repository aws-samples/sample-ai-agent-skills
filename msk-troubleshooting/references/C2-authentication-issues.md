---
title: "C2 — MSK Authentication Issues"
description: "Diagnose and resolve authentication failures across IAM, SASL/SCRAM, and mTLS"
status: active
severity: HIGH
triggers:
  - "authentication failed"
  - "auth error"
  - "SASL authentication"
  - "IAM auth"
  - "SSL handshake"
  - "access denied"
  - "not authorized"
owner: devops-agent
objective: "Identify authentication configuration issues and restore client access"
context: "MSK supports three authentication methods: IAM (port 9098), SASL/SCRAM (port 9096), and mTLS (port 9094). Each requires specific client configuration. Common issues include wrong port, missing IAM policy, expired SCRAM secrets, certificate mismatches, or mixing auth method configurations. A cluster can enable multiple auth methods simultaneously."
---

## Phase 1 — Triage

MUST:
- Check cluster auth configuration: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.Provisioned.ClientAuthentication'`
- Get bootstrap brokers for each auth method: `aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>`
- Identify which auth method the client is using and verify correct port
- Check cluster status: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.State'`

SHOULD:
- For IAM: check IAM policy attached to the client role: `aws iam get-role-policy --role-name <role> --policy-name <policy>` or `aws iam list-attached-role-policies --role-name <role>`
- For SASL/SCRAM: check secrets: `aws kafka list-scram-secrets --cluster-arn <cluster-arn>`
- For mTLS: verify ACM Private CA certificate is valid
- Check CloudTrail for auth-related events

MAY:
- Test auth from a known-good client in the same VPC
- Check client-side logs for specific auth error messages
- Verify client library version supports the auth method

## Phase 2 — Remediate

MUST:
- If wrong port: switch to correct port (9098=IAM, 9096=SASL/SCRAM, 9094=mTLS)
- If IAM policy missing: add `kafka-cluster:Connect`, `kafka-cluster:DescribeCluster`, and topic/group permissions
- If SCRAM secret invalid: update or recreate the secret in Secrets Manager and re-associate
- If certificate expired: renew the ACM Private CA certificate

SHOULD:
- For IAM: ensure client uses `software.amazon.msk.auth.iam.IAMClientCallbackHandler`
- For SASL/SCRAM: verify secret format matches `{"username":"...","password":"..."}`
- For mTLS: ensure client truststore includes the CA certificate
- Test with minimal permissions first, then expand

MAY:
- Enable multiple auth methods for migration between methods
- Use IAM auth for new applications (simplest to manage)
- Set up monitoring for authentication failure metrics

## Common Issues

- symptoms: "SASL authentication failed with SCRAM-SHA-512"
  diagnosis: "SCRAM secret not associated with the cluster or secret format incorrect."
  resolution: "Associate the secret: aws kafka batch-associate-scram-secret. Verify secret JSON format."

- symptoms: "IAM auth fails with 'not authorized to perform kafka-cluster:Connect'"
  diagnosis: "IAM policy missing required kafka-cluster actions."
  resolution: "Add kafka-cluster:Connect, kafka-cluster:DescribeCluster, and topic/group permissions to the IAM policy."

- symptoms: "SSL handshake failure with mTLS"
  diagnosis: "Client certificate not signed by the ACM Private CA associated with the cluster."
  resolution: "Issue client certificate from the correct ACM Private CA. Verify CA ARN in cluster config."

## Output Format

```yaml
root_cause: "authentication_failure — <specific_cause>"
evidence:
  - type: cluster_auth_config
    content: "<enabled auth methods>"
  - type: bootstrap_brokers
    content: "<endpoints per auth method>"
  - type: auth_details
    content: "<IAM policy / SCRAM secrets / certificate status>"
severity: HIGH
mitigation:
  immediate: "Fix auth configuration (port, credentials, certificates)"
  long_term: "Document auth requirements, automate credential rotation"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster auth configuration: GREEN — read-only API call"
  - "Get bootstrap brokers: GREEN — read-only API call"
  - "List SCRAM secrets: GREEN — read-only API call"
  - "Update IAM policy: YELLOW — changes access permissions, may grant unintended access"
  - "Update or recreate SCRAM secret: YELLOW — credential change may disrupt other clients using the same secret"
  - "Associate SCRAM secret with cluster: YELLOW — enables new authentication credential"
```

## Escalation Conditions
- Cluster serves production event streaming
- Authentication failures blocking production applications
- Fix requires IAM policy changes affecting multiple services
- SCRAM secret rotation needed across multiple client applications
- mTLS certificate renewal required before expiry

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "SCRAM credentials: username and password in Secrets Manager"
    - "IAM role ARNs and policies: access control configuration"
    - "mTLS certificates: client authentication credentials"
    - "ACM Private CA ARN: certificate authority identifier"
  handling: "NEVER log or expose SCRAM passwords. Mask IAM role ARNs in shared outputs. Handle certificates as secrets."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER log or display SCRAM secret passwords
- NEVER use wildcard (*) in IAM kafka-cluster policies for production
- NEVER disable authentication on a production cluster
- NEVER share mTLS private keys or CA certificates in logs

## Phase 3 — Rollback
- If IAM policy was modified: revert to previous policy version using IAM policy versioning
- If SCRAM secret was updated: restore previous secret value in Secrets Manager from version history
- If SCRAM secret was associated: batch-disassociate the secret if it was added in error
- If mTLS certificate was renewed: previous certificate remains valid until expiry — no immediate rollback needed
- If authentication method was enabled on cluster: cannot disable auth methods without cluster recreation — plan carefully

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"
