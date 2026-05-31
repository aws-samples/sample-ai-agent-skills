---
title: "C1 — External Access Detection"
description: "Diagnose external access detection issues"
status: active
severity: HIGH
triggers: ["external access not detected", "public resource not found", "missing finding"]
owner: devops-agent
objective: "Resolve external access detection issues"
context: "External access analysis detects resources shared with principals outside the zone of trust (account or organization)."
---

## Phase 1 — Triage

MUST:
- Verify analyzer exists and is ACTIVE in the resource's region
- Check if resource type is supported (S3, IAM roles, KMS, Lambda, SQS, etc.)
- Verify resource has external access in its policy

SHOULD:
- Check if archive rules are hiding the finding
- Verify analyzer type matches scope (ACCOUNT vs ORGANIZATION)

MAY:
- Trigger resource scan: `aws accessanalyzer start-resource-scan --analyzer-arn <arn> --resource-arn <resource-arn>`

## Phase 2 — Remediate

MUST:
- Create analyzer in the resource's region if missing
- Ensure resource type is supported
- Check archive rules aren't hiding findings

SHOULD:
- Wait for analysis cycle or trigger manual scan

MAY:
- Create analyzers in all regions

## Common Issues

- symptoms: "Public S3 bucket but no Access Analyzer finding"
  diagnosis: "No analyzer in the bucket's region or S3 Block Public Access overrides."
  resolution: "Create analyzer in the region. Note: S3 Block Public Access may prevent actual public access even if policy allows it."

## Safety Ratings

```
safety_ratings:
  - "start-resource-scan: GREEN — triggers re-analysis, non-destructive"
  - "create-analyzer in new region: YELLOW — creates new resource with ongoing cost"
  - "list-archive-rules: GREEN — read-only check for suppressed findings"
  - "multi-region analyzer deployment: YELLOW — creates resources across regions"
```

## Escalation Conditions

- Public S3 bucket or Lambda function detected with no corresponding finding
- Archive rules are hiding external access findings for sensitive resources
- Analyzer missing in regions with active resources
- External access detected to resources containing PII or financial data

## Data Sensitivity

- `aws accessanalyzer start-resource-scan`: LOW — triggers analysis only
- Resource policy review: HIGH — reveals access control configuration
- Archive rule filters: MEDIUM — reveals what access patterns are being accepted

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER assume S3 Block Public Access makes a public bucket policy safe without verifying
- NEVER ignore missing analyzer coverage in regions with active resources

## Phase 3 — Rollback

1. If an analyzer was created in the wrong region, delete it: `aws accessanalyzer delete-analyzer --analyzer-name <name> --region <wrong-region>`
2. If archive rules were modified to investigate hidden findings, restore them: `aws accessanalyzer create-archive-rule --analyzer-name <name> --rule-name <rule> --filter <original-filter>`
3. Verify analyzer coverage matches the intended scope across all regions
4. Confirm all known external access patterns have corresponding findings
5. Document any gaps in analyzer coverage for the security team

## Output Format

```yaml
root_cause: "external_access_detection — <specific_cause>"
evidence:
  - type: detection_status
    content: "<analyzer status and resource analysis>"
severity: HIGH
mitigation:
  immediate: "Ensure analyzer coverage"
  long_term: "Deploy analyzers in all regions"
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
