---
title: "A2 — Revision Publishing"
description: "Diagnose revision creation and publishing failures"
status: active
severity: HIGH
triggers: ["revision not finalized", "publish revision failed", "assets not added", "revision stuck in draft"]
owner: devops-agent
objective: "Resolve revision publishing issues including asset addition, finalization, and subscriber visibility"
context: "Revisions are created as drafts, assets are added, then revisions are finalized. Only finalized revisions are visible to subscribers. Revisions cannot be un-finalized."
---

## Phase 1 — Triage

MUST:
- List revisions: `aws dataexchange list-data-set-revisions --data-set-id <dataset-id>`
- Check revision status: `aws dataexchange get-revision --data-set-id <dataset-id> --revision-id <revision-id>`
- List assets in revision: `aws dataexchange list-revision-assets --data-set-id <dataset-id> --revision-id <revision-id>`

SHOULD:
- Check import job status if assets were imported: `aws dataexchange list-jobs --data-set-id <dataset-id>`
- Verify S3 source bucket permissions for S3 snapshot imports

MAY:
- Review CloudTrail for UpdateRevision (finalize) errors
- Check event action configuration for automatic publishing

## Phase 2 — Remediate

MUST:
- Complete pending import jobs before finalizing revision
- Add at least one asset to the revision before finalization
- Finalize revision: `aws dataexchange update-revision --data-set-id <dataset-id> --revision-id <revision-id> --finalized`

SHOULD:
- Verify S3 source objects exist and are accessible
- Check that import job completed successfully before finalizing

MAY:
- Set up event actions for automatic revision publishing
- Configure S3 event notifications for automated workflows

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-revisions/get-revision: GREEN — read-only status check"
  - "dataexchange list-revision-assets: GREEN — read-only asset listing"
  - "dataexchange update-revision --finalized: YELLOW — makes revision visible to subscribers, irreversible"
  - "dataexchange create-job (import): YELLOW — imports data from S3 to revision"
```

## Escalation Conditions

- Import jobs fail repeatedly with internal service errors
- Revision finalization fails despite having valid assets
- Large data sets cause import job timeouts
- Event actions fail to trigger automatic publishing

## Data Sensitivity

- `aws dataexchange list-revisions`: LOW — returns revision metadata
- `aws dataexchange list-revision-assets`: MEDIUM — reveals asset names and S3 paths
- `aws dataexchange get-revision`: LOW — returns revision configuration
- `import job details`: MEDIUM — contains S3 bucket and key information

## Prohibited Actions

- NEVER finalize a revision without verifying all intended assets are included
- NEVER delete a finalized revision with active subscribers without notification
- NEVER import assets from S3 buckets containing sensitive data without classification review
- NEVER bypass the finalization step by modifying revision state directly

## Phase 3 — Rollback

1. If a revision was finalized prematurely, create a new revision with correct assets (finalization is irreversible)
2. If import jobs created incorrect assets, delete the draft revision and start over
3. If event actions published incorrect data, revoke the revision and notify subscribers
4. Verify subscriber access to the correct revision
5. Document the publishing issue and update the publishing workflow

## Common Issues

- symptoms: "Revision finalization fails — no assets in revision"
  diagnosis: "Attempting to finalize a revision with zero assets."
  resolution: "Add assets via import job before finalizing."

## Output Format

```yaml
root_cause: "revision_publishing — <specific_cause>"
evidence:
  - type: revision_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix publishing issue"
  long_term: "Automate revision publishing with event actions"
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
