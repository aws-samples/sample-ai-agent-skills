---
title: "C2 — Source Location Errors"
description: "Diagnose source location creation and access errors"
status: active
severity: HIGH
triggers: ["source location error", "cannot create source location", "VOD source not found", "source access denied"]
owner: devops-agent
objective: "Resolve source location configuration and access issues"
context: "Source locations define HTTP endpoints where MediaTailor retrieves VOD content for channel assembly. They support HTTP and S3 origins with optional SigV4 authentication."
---

## Phase 1 — Triage

MUST:
- List source locations: `aws mediatailor list-source-locations --query 'Items[*].{Name:SourceLocationName,HttpConfiguration:HttpConfiguration}'`
- Describe source location: `aws mediatailor describe-source-location --SourceLocationName <name>`
- List VOD sources: `aws mediatailor list-vod-sources --SourceLocationName <name>`

SHOULD:
- Test HTTP endpoint accessibility: `curl -I "<base-url>/<source-path>"`
- Verify S3 bucket policy allows MediaTailor access if using S3 origin
- Check IAM role for SigV4 access if configured

MAY:
- Review CloudTrail for source location API errors
- Test individual VOD source manifests for validity

## Phase 2 — Remediate

MUST:
- Fix HTTP endpoint URL if unreachable
- Update S3 bucket policy to allow MediaTailor service principal access
- Fix SigV4 IAM role permissions if using authenticated S3 access

SHOULD:
- Verify all VOD sources under the location have valid manifests
- Configure access configuration (SigV4) for private S3 origins

MAY:
- Set up CloudFront distribution in front of S3 for better performance
- Configure segment delivery configurations for CDN integration

## Safety Ratings

```
safety_ratings:
  - "list-source-locations: GREEN — read-only enumeration"
  - "describe-source-location: GREEN — read-only details"
  - "create-source-location: YELLOW — creates new resource"
  - "delete-source-location: RED — removes source location, breaks channels using it"
  - "S3 bucket policy update: YELLOW — modifies access permissions"
```

## Escalation Conditions

- Source location endpoint is down affecting multiple channels
- S3 bucket policy changes require cross-account coordination
- SigV4 role requires permissions changes managed by security team
- Source location deletion needed but referenced by active channels

## Data Sensitivity

- `aws mediatailor list-source-locations`: LOW — returns location metadata
- `aws mediatailor describe-source-location`: MEDIUM — reveals origin URLs and access configuration
- S3 bucket policies: MEDIUM — reveal access permissions and principal ARNs
- VOD source manifests: LOW — content metadata only

## Prohibited Actions

- NEVER delete a source location referenced by active channel programs
- NEVER modify S3 bucket policies to allow public access for MediaTailor
- NEVER share SigV4 IAM role credentials outside the operations team
- NEVER create source locations pointing to non-HTTPS endpoints in production

## Phase 3 — Rollback

1. If source location was deleted, recreate with same name and configuration
2. If S3 bucket policy was changed incorrectly, revert to previous policy
3. If SigV4 role was modified, restore previous IAM role permissions
4. Verify all channels using the source location resume playback after fix

## Common Issues

- symptoms: "Source location returns 403 when MediaTailor tries to access content"
  diagnosis: "S3 bucket policy does not allow MediaTailor service principal or SigV4 role lacks permissions."
  resolution: "Add mediatailor.amazonaws.com to bucket policy or fix SigV4 IAM role permissions."

- symptoms: "VOD source created but channel cannot play it"
  diagnosis: "VOD source manifest URL is invalid or manifest format is unsupported."
  resolution: "Verify VOD source manifest is valid HLS/DASH. Test manifest URL directly."

## Output Format

```yaml
root_cause: "source_location — <specific_cause>"
evidence:
  - type: source_error
    content: "<source location details and error>"
severity: HIGH
mitigation:
  immediate: "Fix source location access"
  long_term: "Implement origin health checks"
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
