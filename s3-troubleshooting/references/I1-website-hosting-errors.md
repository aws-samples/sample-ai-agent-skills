---
title: "I1 — Static Website Hosting Errors"
description: "Diagnose S3 static website hosting configuration and access errors"
status: active
severity: MEDIUM
triggers:
  - "Website hosting not working"
  - "Static website 403"
  - "Index document not found"
  - "S3 website endpoint"
owner: devops-agent
objective: "Fix S3 static website hosting configuration and access issues"
context: "S3 static website hosting uses a different endpoint format: <bucket>.s3-website-<region>.amazonaws.com. It requires an index document, a bucket policy allowing public read (or CloudFront OAC), and Block Public Access must allow the policy. Website hosting does not support HTTPS directly — use CloudFront for HTTPS."
---

## Phase 1 — Triage

MUST:
- Check website configuration: `aws s3api get-bucket-website --bucket <bucket>`
- Verify index document exists: `aws s3api head-object --bucket <bucket> --key index.html`
- Check bucket policy for public read: `aws s3api get-bucket-policy --bucket <bucket>`
- Check Block Public Access: `aws s3api get-public-access-block --bucket <bucket>`

SHOULD:
- Verify the correct website endpoint URL is being used (not the REST API endpoint)
- Check error document configuration
- Verify all referenced files exist in the bucket

MAY:
- Test with curl to the website endpoint: `curl -I http://<bucket>.s3-website-<region>.amazonaws.com`
- Check if CloudFront is configured in front of the website endpoint

## Phase 2 — Remediate

MUST:
- Enable website hosting with index document: `aws s3 website s3://<bucket> --index-document index.html --error-document error.html`
- Add bucket policy for public read (if not using CloudFront):
  ```json
  {
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<bucket>/*"
  }
  ```
- Disable Block Public Access settings that block the public policy (or use CloudFront OAC instead)

SHOULD:
- Use CloudFront with OAC for HTTPS and keep the bucket private
- Configure a custom error document for 404 pages
- Set correct Content-Type on uploaded files

MAY:
- Configure redirect rules for URL routing
- Set up Route 53 alias record for custom domain

## Common Issues

- symptoms: "403 Forbidden on website endpoint"
  diagnosis: "Bucket policy does not allow public read, or Block Public Access is enabled."
  resolution: "Add public read bucket policy and adjust Block Public Access, or use CloudFront OAC."

- symptoms: "404 Not Found on website root"
  diagnosis: "Index document is not configured or the file does not exist."
  resolution: "Set the index document and upload the file (e.g., index.html)."

- symptoms: "Website works with REST endpoint but not website endpoint"
  diagnosis: "Website hosting is not enabled on the bucket."
  resolution: "Enable static website hosting in the bucket properties."

## Output Format

```yaml
root_cause: "website_hosting — <specific_cause>"
evidence:
  - type: website_config
    content: "<website configuration>"
  - type: bucket_policy
    content: "<public access policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix website configuration and access policy"
  long_term: "Use CloudFront with OAC for HTTPS and caching"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🔴 RED | Involves modifying Block Public Access settings and adding public bucket policies via put-public-access-block and put-bucket-policy with Principal "*". Website hosting inherently requires public access unless CloudFront OAC is used. Highest-risk S3 security operations. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Public access changes expose bucket contents to the internet

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Website configuration reveals hosting architecture"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
