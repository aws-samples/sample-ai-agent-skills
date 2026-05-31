---
name: healthimaging-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS HealthImaging
  problems by analyzing datastore creation, DICOM import jobs,
  image set retrieval, copy/deletion operations, access control,
  encryption, and integration issues using structured runbooks.
  Activate when: datastore creation failures, import job errors,
  DICOM validation problems, image set retrieval issues, copy job
  failures, deletion errors, access control problems, encryption
  configuration issues, or the user says something is wrong with
  HealthImaging.
compatibility: >
  Requires AWS CLI or SDK access with medical-imaging, s3, iam,
  kms, cloudtrail, and cloudwatch permissions.
---

# AWS HealthImaging Diagnostics

## When to use

Any AWS HealthImaging investigation — datastore creation, DICOM import jobs, image set retrieval, metadata queries, copy/deletion operations, access control, encryption configuration, or integration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws medical-imaging list-datastores --query 'datastoreSummaries[*].{Name:datastoreName,Id:datastoreId,Status:datastoreStatus}'
aws medical-imaging list-dicom-import-jobs --datastore-id <datastore-id> --query 'jobSummaries[*].{JobId:jobId,JobName:jobName,Status:jobStatus}'
aws medical-imaging search-image-sets --datastore-id <datastore-id> --search-criteria '{"filters":[{"operator":"EQUAL","values":[{"DICOMPatientId":"<patient-id>"}]}]}'
```

### Step 2 — Domain deep dive

```
aws medical-imaging get-datastore --datastore-id <datastore-id>
aws medical-imaging get-dicom-import-job --datastore-id <datastore-id> --job-id <job-id>
aws medical-imaging get-image-set --datastore-id <datastore-id> --image-set-id <image-set-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medical-imaging.amazonaws.com --max-results 20
aws medical-imaging get-image-set-metadata --datastore-id <datastore-id> --image-set-id <image-set-id>
aws medical-imaging get-image-frame --datastore-id <datastore-id> --image-set-id <image-set-id> --image-frame-information '{"imageFrameId":"<frame-id>"}' outfile.jph
```

Read `references/guardrails.md` before concluding on any HealthImaging issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `medical-imaging list-datastores` | List datastores |
| `medical-imaging get-datastore` | Get datastore details |
| `medical-imaging list-dicom-import-jobs` | List import jobs |
| `medical-imaging get-dicom-import-job` | Get import job details |
| `medical-imaging search-image-sets` | Search image sets |
| `medical-imaging get-image-set` | Get image set details |
| `medical-imaging get-image-set-metadata` | Get DICOM metadata |

## Gotchas: AWS HealthImaging

- HealthImaging stores DICOM data in datastores. Each datastore is an isolated container — image sets cannot be shared across datastores without copy operations.
- DICOM import jobs read from S3 and validate DICOM compliance. Invalid DICOM files are rejected per-file, not per-job — partial imports are possible.
- Image sets are immutable versioned collections. Updates create new versions; previous versions are retained until explicitly deleted.
- Metadata queries use DICOM tag-based search criteria. Not all DICOM tags are indexed — only patient, study, and series level tags are searchable.
- Copy operations are asynchronous and cross-datastore. They create independent copies — changes to the source do not propagate to the copy.
- Deletion is a two-step process: delete the image set, then optionally purge. Deleted image sets may be recoverable within a retention window.
- HealthImaging supports AWS KMS encryption. Datastores use either AWS-managed or customer-managed KMS keys configured at creation time and cannot be changed later.

## Anti-hallucination rules

1. Always cite specific datastore IDs, job IDs, or image set IDs as evidence.
2. DICOM import failures can be partial — never assume the entire job failed from a single file error.
3. Image set versions are immutable. Never claim in-place modification is possible.
4. Metadata search is limited to indexed DICOM tags. Never claim arbitrary tag search is supported.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Datastore | A1-A2 | Creation, configuration |
| B — Import | B1-B2 | Import job failures, DICOM validation errors |
| C — Retrieval | C1-C2 | Image set retrieval, metadata queries |
| D — Operations | D1-D2 | Copy job failures, deletion issues |
| E — Security | E1-E2 | Access control, encryption configuration |
| F — Integration | F1 | Integration issues |
| Z — Catch-All | Z1 | General troubleshooting |
