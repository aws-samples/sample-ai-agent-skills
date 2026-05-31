---
title: "B1 — Audience Generation Issues"
description: "Diagnose audience generation job failures and output problems"
status: active
severity: HIGH
triggers: ["audience generation failed", "generation job error", "no audience output", "job stuck"]
owner: devops-agent
objective: "Resolve audience generation job failures and ensure correct output delivery"
context: "Audience generation jobs consume privacy budget and produce audience segments in S3. Jobs are asynchronous and require configured audience models associated with collaborations."
---

## Phase 1 — Triage

MUST:
- List audience generation jobs: `aws cleanroomsml list-audience-generation-jobs --query 'audienceGenerationJobs[*].{Name:name,Arn:audienceGenerationJobArn,Status:status,CreateTime:createTime}'`
- Get failing job details: `aws cleanroomsml get-audience-generation-job --audience-generation-job-arn <arn>`
- Check privacy budget remaining: verify budget has not been exhausted for the collaboration

SHOULD:
- Verify configured audience model association is active
- Check seed audience S3 data is accessible: `aws s3 ls s3://<seed-bucket>/<prefix>/`

MAY:
- Review CloudTrail for StartAudienceGenerationJob errors
- Check CloudWatch for job-level metrics

## Phase 2 — Remediate

MUST:
- Ensure privacy budget has remaining capacity before retrying
- Fix seed audience data format and size issues
- Verify output S3 bucket permissions allow Clean Rooms ML to write

SHOULD:
- Validate seed audience meets minimum size requirements (500+ records)
- Confirm KMS key grants encrypt permission to the service role

MAY:
- Run a smaller test audience generation job to validate configuration
- Adjust audience size parameters if output was unexpectedly small

## Common Issues

- symptoms: "Audience generation job status is FAILED"
  diagnosis: "Privacy budget exhausted or seed audience below minimum size."
  resolution: "Check privacy budget consumption. Verify seed audience has 500+ records. Wait for budget refresh if exhausted."

- symptoms: "Job completes but output is empty or missing"
  diagnosis: "Output S3 bucket permissions prevent writing or KMS encryption fails."
  resolution: "Grant s3:PutObject and kms:GenerateDataKey to the service role on the output bucket."

## Safety Ratings

```
safety_ratings:
  - "list-audience-generation-jobs: GREEN — read-only enumeration"
  - "get-audience-generation-job: GREEN — read-only detail retrieval"
  - "start-audience-generation-job: YELLOW — consumes privacy budget, creates output data"
  - "s3:PutObject on output bucket: YELLOW — writes audience data to S3"
```

## Escalation Conditions

- Audience generation jobs fail consistently with internal service errors
- Privacy budget is exhausted and business requires immediate audience generation
- Output data appears corrupted or significantly smaller than expected
- Job runs for over 4 hours without status change

## Data Sensitivity

- `aws cleanroomsml get-audience-generation-job`: MEDIUM — reveals job configuration and seed audience details
- `aws s3 ls s3://<output-bucket>/`: MEDIUM — reveals output file names
- `aws cleanroomsml start-audience-generation-job`: HIGH — consumes privacy budget and generates audience data
- Audience output data: HIGH — contains derived audience segments

## Prohibited Actions

- NEVER start audience generation jobs without confirming privacy budget availability
- NEVER access or download generated audience data without authorization from all collaboration members
- NEVER retry failed jobs repeatedly without investigating root cause — each attempt consumes privacy budget
- NEVER share audience generation output paths with parties outside the collaboration

## Phase 3 — Rollback

1. Failed jobs do not produce output — no data cleanup needed
2. If privacy budget was consumed by a failed job, document the consumption for budget tracking
3. If output was written to wrong S3 location, delete the misplaced output: `aws s3 rm s3://<bucket>/<prefix>/ --recursive` (with data owner approval)
4. Verify no downstream systems consumed incorrect audience data
5. Notify collaboration members of job failure and budget impact

## Output Format

```yaml
root_cause: "audience_generation_failure — <specific_cause>"
evidence:
  - type: job_error
    content: "<error details from get-audience-generation-job>"
severity: HIGH
mitigation:
  immediate: "Fix root cause and retry with budget verification"
  long_term: "Implement pre-flight checks before audience generation"
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
