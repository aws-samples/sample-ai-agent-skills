---
title: "A2 — ID Mapping Workflow Errors"
description: "Diagnose Entity Resolution ID mapping workflow failures including provider connectivity, configuration, and data format errors"
status: active
severity: HIGH
triggers: ["ID mapping failed", "id mapping workflow error", "identity mapping not working", "StartIdMappingJob error"]
owner: devops-agent
objective: "Resolve ID mapping workflow execution failures"
context: "ID mapping workflows map identifiers across datasets using provider services. They are separate from matching workflows and use different APIs and configuration. Provider service availability affects ID mapping."
---

## Phase 1 — Triage

MUST:
- Get workflow details: `aws entityresolution get-id-mapping-workflow --workflow-name <workflow-name>`
- Check job status: `aws entityresolution get-id-mapping-job --workflow-name <workflow-name> --job-id <job-id>`
- Verify provider service configuration

SHOULD:
- Check the service role permissions: `aws iam get-role --role-name <role-name>`
- Verify input data source accessibility
- Review CloudTrail for ID mapping events

MAY:
- Check provider service status independently
- Verify input data format matches provider requirements
- Review CloudWatch metrics for ID mapping jobs

## Phase 2 — Remediate

MUST:
- Fix provider service configuration and credentials
- Correct service role permissions for input/output access
- Ensure input data format matches provider requirements

SHOULD:
- Verify provider subscription is active
- Test with a small dataset before full execution
- Check provider-specific error codes and documentation

MAY:
- Set up monitoring for ID mapping job failures
- Implement retry logic for transient provider errors

## Safety Ratings

```
safety_ratings:
  - "get-id-mapping-workflow: GREEN — read-only workflow configuration"
  - "get-id-mapping-job: GREEN — read-only job status"
  - "start-id-mapping-job: YELLOW — initiates ID mapping processing"
  - "delete-id-mapping-workflow: RED — permanently removes workflow"
```

## Escalation Conditions

- Provider service is unavailable and ID mapping is business-critical
- ID mapping produces incorrect mappings affecting downstream systems
- Provider-specific errors require vendor support engagement
- Cross-account ID mapping fails due to organizational policies

## Data Sensitivity

- `aws entityresolution get-id-mapping-workflow`: LOW — workflow configuration
- `aws entityresolution get-id-mapping-job`: LOW — job metadata
- Input data: HIGH — contains identifiers that may be PII
- Output mappings: HIGH — contains cross-referenced identifiers

## Prohibited Actions

- NEVER share provider credentials or API keys in logs or chat
- NEVER run ID mapping on production data without testing
- NEVER ignore provider service terms of use
- NEVER store ID mapping output in unencrypted locations

## Phase 3 — Rollback

1. If ID mapping produced incorrect results, delete output data
2. If the workflow was misconfigured, update with correct settings
3. If provider credentials were changed, restore previous configuration
4. Notify downstream systems about incorrect ID mappings
5. Document the failure and coordinate with provider if needed

## Common Issues

- symptoms: "ID mapping job fails with provider error"
  diagnosis: "Provider service subscription is inactive, credentials are invalid, or the provider is experiencing an outage."
  resolution: "Verify provider subscription status. Check credentials. Contact provider support if the service is down."

- symptoms: "ID mapping returns no results"
  diagnosis: "Input identifiers do not match any records in the provider's database, or input format is incorrect."
  resolution: "Verify input data format matches provider requirements. Test with known identifiers."

- symptoms: "ID mapping job fails with AccessDeniedException"
  diagnosis: "The service role lacks permissions for input data or output location."
  resolution: "Grant the service role permissions for Glue, S3, and KMS resources used by the workflow."

## Output Format

```yaml
root_cause: "id_mapping_workflow — <specific_cause>"
evidence:
  - type: id_mapping_error
    content: "<job failure details>"
severity: HIGH
mitigation:
  immediate: "Fix ID mapping configuration"
  long_term: "Implement provider health monitoring"
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
