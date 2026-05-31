---
title: "E3 — Field-Level Encryption Issues"
description: "Diagnose field-level encryption configuration and decryption failures"
status: active
severity: MEDIUM
triggers:
  - "field-level encryption"
  - "FLE"
  - "field encryption"
  - "encrypted field"
owner: devops-agent
objective: "Resolve field-level encryption configuration and operational issues"
context: "CloudFront field-level encryption encrypts specific POST body fields at the edge using RSA public keys. Only the application with the corresponding private key can decrypt. Limited to 10 fields per profile, 1 MB total POST body. Uses OAEP padding with SHA-256. The encrypted data is base64-encoded and replaces the original field value."
---

## Phase 1 — Triage

MUST:
- Check FLE configuration: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.FieldLevelEncryptionId'`
- Get FLE profile: `aws cloudfront get-field-level-encryption-profile --id <profile-id>`
- Get FLE config: `aws cloudfront get-field-level-encryption --id <fle-id>`
- Verify the public key is correctly configured: `aws cloudfront get-public-key --id <key-id>`

SHOULD:
- Verify the POST body content type matches the FLE configuration (application/x-www-form-urlencoded or multipart/form-data)
- Check that field names in the FLE profile match the actual POST body field names
- Verify the application can decrypt with the corresponding private key

MAY:
- Test with a sample POST request to verify encryption
- Check CloudFront error logs for FLE-related errors

## Phase 2 — Remediate

MUST:
- Ensure public key is valid RSA key (2048-bit minimum)
- Match field names exactly (case-sensitive) between FLE profile and POST body
- Ensure POST body is under 1 MB and has no more than 10 encrypted fields

SHOULD:
- Use separate key pairs for different environments
- Implement key rotation procedures
- Test decryption in the application before deploying

MAY:
- Monitor FLE errors in CloudWatch
- Implement fallback handling for decryption failures

## Common Issues

- symptoms: "POST request returns 400 Bad Request with FLE enabled"
  diagnosis: "POST body exceeds 1 MB or content type is not supported."
  resolution: "Ensure POST body < 1 MB and content type is form-urlencoded or multipart."

- symptoms: "Field not encrypted in the POST body"
  diagnosis: "Field name in FLE profile does not match the actual POST field name."
  resolution: "Verify field names are exact matches (case-sensitive)."

- symptoms: "Application cannot decrypt the field"
  diagnosis: "Wrong private key or incorrect decryption padding."
  resolution: "Use the private key matching the public key in FLE config. Use OAEP with SHA-256."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
```

## Escalation Conditions

- Distribution serves a production website or application
- Fix requires modifying origin configuration or cache behaviors
- Resolution involves certificate changes or HTTPS configuration
- Issue affects multiple distributions or is account-level
- Lambda@Edge or CloudFront Functions changes are required on production

## Data Sensitivity

MEDIUM - Signed URL private keys and key pairs control content access. Origin configurations may expose internal infrastructure (S3 bucket names, ALB endpoints). Access logs contain client IPs, request URIs, and query strings. Field-level encryption configurations protect sensitive form data.

## Prohibited Actions

- NEVER suggest deleting a CloudFront distribution that is serving live traffic
- NEVER suggest disabling HTTPS or downgrading the security policy on a production distribution
- NEVER recommend removing all cache behaviors - this breaks content routing
- NEVER suggest invalidating '/*' repeatedly as a fix - address the root caching issue instead
- NEVER recommend removing origin access control/identity from S3 origins without alternative access controls

## Phase 3 - Rollback

1. If distribution configuration was changed, update with previous settings: `aws cloudfront update-distribution --id <id> --distribution-config <previous> --if-match <etag>`
2. If cache policy or TTL was changed, restore original cache behavior settings and allow caches to repopulate
3. If access restriction settings were changed, restore original trusted key groups or signers
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "field_level_encryption — <specific_cause>"
evidence:
  - type: fle_config
    content: "<FLE profile and field mappings>"
  - type: public_key
    content: "<public key configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix FLE configuration or key mismatch"
  long_term: "Implement key rotation and decryption testing"
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
  - "NEVER suggest disabling HTTPS requirements"
  - "NEVER suggest removing WAF association to fix access"
  - "NEVER suggest wildcard CORS origins in production"
