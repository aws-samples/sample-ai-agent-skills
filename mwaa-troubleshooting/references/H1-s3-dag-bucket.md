---
title: "H1 — MWAA S3 DAG Bucket Issues"
description: "Diagnose S3 DAG bucket sync and access problems in MWAA"
status: active
severity: HIGH
triggers:
  - "DAGs not syncing"
  - "S3 bucket error"
  - "DAG bucket"
  - "S3 access denied"
  - "DAG files not updating"
owner: devops-agent
objective: "Identify and resolve S3 DAG bucket access and sync issues in MWAA"
context: "MWAA syncs DAGs from a configured S3 bucket path. The bucket must have versioning enabled, the execution role must have read access, and the bucket must be in the same region as the environment."
---

## Phase 1 — Triage

MUST:
- Check S3 bucket configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.{SourceBucketArn:SourceBucketArn,DagS3Path:DagS3Path}'`
- Verify bucket exists and is accessible: `aws s3 ls s3://<bucket>/<dags-path>/`
- Check bucket versioning: `aws s3api get-bucket-versioning --bucket <bucket-name>`
- Verify execution role S3 permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject s3:ListBucket --resource-arns arn:aws:s3:::<bucket>`
- Check bucket policy for deny rules: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Verify bucket is in the same region as the MWAA environment
- Check if bucket encryption requires KMS permissions
- Verify no S3 Block Public Access settings interfere (shouldn't for MWAA)
- Check DAG file sizes and counts: `aws s3 ls s3://<bucket>/<dags-path>/ --recursive --summarize`

MAY:
- Check S3 access logs for denied requests
- Verify no lifecycle rules are deleting DAG files prematurely

## Phase 2 — Remediate

MUST:
- Enable bucket versioning if disabled: `aws s3api put-bucket-versioning --bucket <bucket-name> --versioning-configuration Status=Enabled`
- Grant execution role `s3:GetObject`, `s3:GetObjectVersion`, `s3:ListBucket` permissions
- Ensure bucket is in the same AWS region as the MWAA environment

SHOULD:
- Grant KMS decrypt permission if bucket uses CMK encryption
- Remove overly restrictive bucket policies that block the execution role
- Organize DAGs in the configured path (not in subdirectories unless intended)

MAY:
- Set up S3 event notifications for DAG file changes
- Implement CI/CD pipeline for DAG deployment to S3

## Common Issues

- symptoms: "DAGs not appearing after upload to S3"
  diagnosis: "Files uploaded to wrong path or bucket versioning disabled."
  resolution: "Verify the DagS3Path and enable bucket versioning."

- symptoms: "S3 access denied errors in scheduler logs"
  diagnosis: "Execution role lacks S3 read permissions for the DAG bucket."
  resolution: "Add s3:GetObject and s3:ListBucket permissions to the execution role."

- symptoms: "DAGs sync but changes not reflected"
  diagnosis: "S3 sync interval delay or cached DAG version."
  resolution: "Wait for the sync cycle (up to 60 seconds). Check DAGProcessing logs for parse confirmation."

## Output Format

```yaml
root_cause: "s3_dag_bucket_issue — <specific_cause>"
evidence:
  - type: bucket_config
    content: "<bucket ARN, DAG path, versioning status>"
  - type: iam_permissions
    content: "<execution role S3 permissions>"
  - type: bucket_policy
    content: "<bucket policy if restrictive>"
severity: HIGH
mitigation:
  immediate: "Fix S3 permissions or bucket configuration"
  long_term: "Automate DAG deployment with CI/CD and S3 versioning"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — S3 bucket contains DAG code and configuration files
- Service Disruption Risk: HIGH — S3 bucket issues prevent DAG synchronization
- Compliance Impact: MEDIUM — S3 bucket security affects code integrity

## Escalation Conditions
- S3 bucket becomes inaccessible to MWAA environment
- Unauthorized modifications detected in the DAG bucket
- S3 bucket policy changes break MWAA access
- Bucket versioning disabled on production DAG bucket
- Large number of DAG files causing sync performance issues

## Data Sensitivity
- DAG source code — INTERNAL
- Plugin archives — INTERNAL
- Requirements files — INTERNAL
- S3 bucket policies — INTERNAL
- S3 access logs — INTERNAL
- Never expose S3 bucket names in public documentation

## Prohibited Actions
- Do NOT make the DAG bucket publicly accessible
- Do NOT disable versioning on the DAG bucket
- Do NOT delete the DAG bucket while an environment is active
- Do NOT grant write access to the DAG bucket to untrusted principals
- Do NOT store non-DAG data in the MWAA DAG bucket path
- Do NOT bypass S3 server-side encryption for the DAG bucket

## Phase 3 — Rollback
- Restore deleted DAG files from S3 versioned objects
- Revert S3 bucket policy to previous configuration
- Re-enable versioning if accidentally disabled
- Restore bucket encryption settings
- Verify MWAA can sync DAGs after bucket configuration changes
- Document S3 bucket changes and update access control policies

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"
