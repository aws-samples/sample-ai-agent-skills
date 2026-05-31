---
title: "C1 — Image Set Retrieval"
description: "Diagnose image set retrieval and frame download issues"
status: active
severity: HIGH
triggers: ["cannot retrieve image set", "get-image-frame error", "image set not found", "frame download failed"]
owner: devops-agent
objective: "Resolve image set retrieval and frame download failures"
context: "Image frames are returned in HTJ2K format. Image sets are versioned and immutable."
---

## Phase 1 — Triage

MUST:
- Verify image set exists: `aws medical-imaging get-image-set --datastore-id <datastore-id> --image-set-id <image-set-id>`
- Check image set status and version
- Verify IAM permissions for medical-imaging:GetImageSet and medical-imaging:GetImageFrame

SHOULD:
- Get image set metadata to find frame IDs: `aws medical-imaging get-image-set-metadata --datastore-id <datastore-id> --image-set-id <image-set-id>`
- Check if image set has been deleted or is in a non-ACTIVE state

MAY:
- Test frame retrieval: `aws medical-imaging get-image-frame --datastore-id <datastore-id> --image-set-id <image-set-id> --image-frame-information '{"imageFrameId":"<frame-id>"}' outfile.jph`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for retrieval operations
- Use correct image set version ID for versioned access
- Ensure client supports HTJ2K decoding for frame data

SHOULD:
- Implement retry logic for transient retrieval errors
- Cache frequently accessed metadata client-side

MAY:
- Use batch retrieval patterns for multi-frame image sets
- Implement client-side DICOM reconstruction from metadata + frames

## Safety Ratings

```
safety_ratings:
  - "get-image-set: GREEN — read-only image set metadata"
  - "get-image-set-metadata: GREEN — read-only DICOM metadata retrieval"
  - "get-image-frame: GREEN — read-only pixel data retrieval"
  - "IAM policy changes: YELLOW — modifies access permissions"
```

## Escalation Conditions

- Image set retrieval consistently fails with internal service errors
- Frame retrieval returns corrupted or incomplete data
- Retrieval latency exceeds clinical workflow requirements
- Large multi-frame image sets (CT/MR) timeout during retrieval

## Data Sensitivity

- `aws medical-imaging get-image-set`: LOW — returns image set metadata
- `aws medical-imaging get-image-set-metadata`: HIGH — contains DICOM metadata with PHI
- `aws medical-imaging get-image-frame`: HIGH — contains medical imaging pixel data
- Frame output files: HIGH — medical imaging data subject to HIPAA

## Prohibited Actions

- NEVER store retrieved PHI data (metadata or frames) in unencrypted locations
- NEVER share frame data or metadata outside authorized systems
- NEVER bypass access controls to retrieve image sets from other datastores
- NEVER cache PHI metadata in application logs

## Phase 3 — Rollback

1. If IAM permissions were broadened for troubleshooting: revert to least-privilege
2. If test frame files were downloaded: securely delete them after troubleshooting
3. If client-side caches contain PHI: clear caches and verify deletion
4. Document all PHI data access during troubleshooting for audit compliance
5. Verify retrieval is functioning normally before closing the investigation

## Common Issues

- symptoms: "GetImageSet returns ResourceNotFoundException"
  diagnosis: "Image set ID is incorrect or image set has been deleted."
  resolution: "Verify image set ID. Search for the image set: aws medical-imaging search-image-sets --datastore-id <id>."

- symptoms: "GetImageFrame returns garbled data"
  diagnosis: "Client is not decoding HTJ2K format correctly."
  resolution: "Ensure client uses HTJ2K decoder. Frame data is High-Throughput JPEG 2000, not standard JPEG or raw pixels."

## Output Format

```yaml
root_cause: "image_set_retrieval — <specific_cause>"
evidence:
  - type: retrieval_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix retrieval issue"
  long_term: "Implement robust retrieval with retry logic"
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
