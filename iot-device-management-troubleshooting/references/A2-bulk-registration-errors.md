---
title: "A2 — Bulk Registration Errors"
description: "Diagnose bulk thing registration task failures and per-line errors"
status: active
severity: MEDIUM
triggers: ["bulk registration failed", "register things task error", "S3 input file rejected", "bulk import errors"]
owner: devops-agent
objective: "Resolve bulk registration failures and ensure all devices are registered"
context: "Bulk registration uses an S3 JSON file to register multiple things. Each line is processed independently. Errors are logged to a separate S3 object."
---

## Phase 1 — Triage

MUST:
- List registration tasks: `aws iot list-thing-registration-tasks --query 'taskIds[*]'`
- Describe the failing task: `aws iot describe-thing-registration-task --task-id <task-id>`
- Check task status and error counts: `aws iot describe-thing-registration-task --task-id <task-id> --query '{Status:status,SuccessCount:successCount,FailureCount:failureCount,InputFileBucket:inputFileBucket,InputFileKey:inputFileKey}'`

SHOULD:
- Download and inspect the input file: `aws s3 cp s3://<bucket>/<key> input-file.json`
- Check the error log if failures exist: review the S3 error log path from task description
- Verify IAM role used for bulk registration: `aws iam get-role --role-name <role-name>`

MAY:
- Validate JSON syntax of input file lines: `python -c "import json; [json.loads(l) for l in open('input-file.json')]"`
- Check S3 bucket permissions for IoT service access
- Review CloudTrail for StartThingRegistrationTask events

## Phase 2 — Remediate

MUST:
- Fix malformed JSON lines in the input file
- Ensure each line contains required fields: thingName and optionally thingTypeName, attributePayload, thingGroupNames
- Verify S3 bucket policy allows IoT service to read the input file and write error logs

SHOULD:
- Validate thing names conform to IoT naming rules (alphanumeric, hyphens, underscores, max 128 chars)
- Ensure referenced thing types exist before bulk registration
- Ensure referenced thing groups exist before bulk registration

MAY:
- Split large input files into smaller batches for easier error isolation
- Implement pre-validation script to check input file before submission

## Safety Ratings

```
safety_ratings:
  - "list-thing-registration-tasks: GREEN — read-only enumeration"
  - "describe-thing-registration-task: GREEN — read-only inspection"
  - "s3 cp (download input): GREEN — read-only download"
  - "start-thing-registration-task: YELLOW — creates things in bulk, verify input file first"
```

## Escalation Conditions

- Bulk registration task stuck in IN_PROGRESS for more than 1 hour
- Error rate exceeds 50% of input lines
- S3 bucket permissions cannot be modified due to bucket policy restrictions
- Registration task creates duplicate things conflicting with existing fleet

## Data Sensitivity

- `aws iot describe-thing-registration-task`: LOW — returns task metadata
- `aws s3 cp` (input file): MEDIUM — may contain device identifiers and attributes
- Error log S3 object: MEDIUM — contains device names and failure reasons
- `aws iam get-role`: LOW — returns role metadata

## Prohibited Actions

- NEVER start a bulk registration task without validating the input file format
- NEVER delete the S3 input file or error log before analyzing failures
- NEVER re-run bulk registration without deduplicating against already-registered things
- NEVER use a bulk registration IAM role with broader permissions than required for thing creation

## Phase 3 — Rollback

1. If incorrect things were registered, list and delete them: `aws iot delete-thing --thing-name <name>` (after detaching principals)
2. If wrong thing type was assigned, update things: `aws iot update-thing --thing-name <name> --remove-thing-type`
3. If wrong group membership, remove from group: `aws iot remove-thing-from-thing-group --thing-group-name <group> --thing-name <name>`
4. Preserve error logs for audit before cleanup
5. Document which things were rolled back and notify the device management team

## Common Issues

- symptoms: "Task status is Failed with 'Access Denied' error"
  diagnosis: "IAM role does not have permission to read the S3 input file or create things."
  resolution: "Update IAM role with s3:GetObject on the input bucket and iot:CreateThing, iot:AddThingToThingGroup permissions."

- symptoms: "Task completes but failureCount is high"
  diagnosis: "Individual lines have JSON formatting errors or reference non-existent thing types/groups."
  resolution: "Download error log from S3. Fix per-line errors in input file. Re-run with corrected file."

## Output Format

```yaml
root_cause: "bulk_registration — <specific_cause>"
evidence:
  - type: registration_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix input file and re-run registration"
  long_term: "Implement input validation pipeline before bulk registration"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
