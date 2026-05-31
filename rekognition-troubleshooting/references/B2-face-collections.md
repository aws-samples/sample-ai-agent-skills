---
title: "B2 — Face Collection Issues"
description: "Diagnose face collection management and search problems"
status: active
severity: HIGH
triggers:
  - "face collection error"
  - "IndexFaces failed"
  - "SearchFaces not matching"
  - "collection not found"
owner: devops-agent
objective: "Identify and resolve face collection issues"
context: "Face collections store face vectors for search. IndexFaces adds faces, SearchFacesByImage searches. Collections are region-specific with default 20M face limit."
---

## Phase 1 — Triage

MUST:
- List collections: `aws rekognition list-collections`
- Describe collection: `aws rekognition describe-collection --collection-id <collection-id>`
- Check face count in collection
- Test search: `aws rekognition search-faces-by-image --collection-id <collection-id> --image '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --face-match-threshold 80`

SHOULD:
- Verify collection is in correct region
- Check IndexFaces quality filter settings
- Review search threshold (FaceMatchThreshold)

MAY:
- List faces in collection: `aws rekognition list-faces --collection-id <collection-id> --max-results 10`
- Check for duplicate faces

## Phase 2 — Remediate

MUST:
- Create collection in correct region
- Use quality images for IndexFaces
- Set appropriate FaceMatchThreshold for searches

SHOULD:
- Use QualityFilter=AUTO for IndexFaces to skip low-quality faces
- Associate ExternalImageId with indexed faces for identification

MAY:
- Implement collection management automation

## Common Issues

- symptoms: "SearchFacesByImage returns no matches for known face"
  diagnosis: "FaceMatchThreshold too high or face indexed with poor quality."
  resolution: "Lower threshold. Re-index with better quality image."

- symptoms: "ResourceNotFoundException for collection"
  diagnosis: "Collection doesn't exist in this region."
  resolution: "Create collection or switch to correct region."

## Output Format

```yaml
root_cause: "face_collection — <specific_cause>"
evidence:
  - type: collection_info
    content: "<collection details>"
  - type: search_result
    content: "<search results>"
severity: HIGH
mitigation:
  immediate: "Fix collection or search configuration"
  long_term: "Implement face collection management and quality controls"
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

- Images and video may contain biometric data (faces, persons)
- Analysis results may contain personally identifiable information
- Custom Labels training data may contain proprietary content
- Classification: HIGHLY CONFIDENTIAL - biometric and PII data requires special handling

## Prohibited Actions

- NEVER delete face collections containing production data
- NEVER stop running Custom Labels models without confirming downstream impact
- NEVER modify content moderation thresholds in production without approval
- NEVER process PII-containing images without confirming data handling compliance

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
