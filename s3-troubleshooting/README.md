# S3 Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon S3 problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for S3 when the console alone isn't enough — access denied analysis, bucket policy evaluation, cross-account access, replication failures, lifecycle rule debugging, encryption issues, performance throttling, event notifications, static website hosting, and Object Lock compliance.

### Activate When

- Access denied errors on S3 operations
- Bucket policy conflicts or unexpected denials
- Cross-account access failures
- CORS errors in browser-based applications
- Replication not working or lagging
- Lifecycle rules not transitioning or expiring objects
- Versioning confusion (delete markers, version management)
- Encryption errors (SSE-S3, SSE-KMS, cross-account KMS)
- Performance throttling (503 SlowDown)
- Event notification failures
- Presigned URL issues
- Static website hosting problems
- Multipart upload failures
- Object Lock or compliance issues

---

## Skill Structure

```
s3-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-access-denied.md
    ├── A2-bucket-policy-conflicts.md
    ├── A3-cross-account-access.md
    ├── A4-vpc-endpoint-access.md
    ├── B1-block-public-access.md
    ├── B2-acl-issues.md
    ├── B3-public-bucket-policy.md
    ├── C1-sse-kms-errors.md
    ├── C2-cross-account-kms.md
    ├── C3-client-side-encryption.md
    ├── D1-replication-failures.md
    ├── D2-cross-account-replication.md
    ├── D3-replication-lag.md
    ├── E1-transition-failures.md
    ├── E2-expiration-issues.md
    ├── E3-abort-multipart.md
    ├── F1-delete-marker-confusion.md
    ├── F2-version-management.md
    ├── F3-mfa-delete.md
    ├── G1-throttling.md
    ├── G2-transfer-acceleration.md
    ├── G3-multipart-upload.md
    ├── H1-event-notification-failures.md
    ├── H2-lambda-triggers.md
    ├── H3-sqs-sns-delivery.md
    ├── I1-website-hosting-errors.md
    ├── I2-cors-failures.md
    ├── I3-cloudfront-integration.md
    ├── J1-retention-issues.md
    ├── J2-legal-hold.md
    ├── Z1-general-troubleshooting.md
    ├── s3-guardrails.md
    └── s3-hallucination-patterns.yaml
```

---

## Runbook Library (34 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Access & Permissions** | A1–A4 | Access denied, bucket policy, cross-account, VPC endpoint |
| **B — Public Access** | B1–B3 | Block public access, ACLs, public bucket policy |
| **C — Encryption** | C1–C3 | SSE-KMS errors, cross-account KMS, client-side |
| **D — Replication** | D1–D3 | Replication failures, cross-account, lag |
| **E — Lifecycle** | E1–E3 | Transition failures, expiration, abort multipart |
| **F — Versioning** | F1–F3 | Delete markers, version management, MFA delete |
| **G — Performance** | G1–G3 | Throttling, transfer acceleration, multipart upload |
| **H — Events & Integration** | H1–H3 | Event notifications, Lambda triggers, SQS/SNS |
| **I — Static Hosting & CORS** | I1–I3 | Website hosting, CORS, CloudFront integration |
| **J — Object Lock** | J1–J2 | Retention, legal hold |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
