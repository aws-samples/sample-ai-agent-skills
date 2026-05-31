---
title: "A2 — Custom Entity Recognizer Issues"
description: "Diagnose custom entity recognizer training and inference problems"
status: active
severity: HIGH
triggers:
  - "custom entity training failed"
  - "entity recognizer error"
  - "custom entities not detected"
owner: devops-agent
objective: "Identify and resolve custom entity recognizer issues"
context: "Custom entity recognizers detect domain-specific entities. Require annotated training data. Must create endpoint for inference. Training data format is critical."
---

## Phase 1 — Triage

MUST:
- Check recognizer status: `aws comprehend describe-entity-recognizer --entity-recognizer-arn <arn>`
- Verify training data format and location in S3
- Check endpoint status: `aws comprehend list-endpoints --filter '{"ModelArn":"<recognizer-arn>"}'`
- Review training metrics (precision, recall, F1)

SHOULD:
- Verify annotation format matches expected schema
- Check minimum training data requirements
- Test with known positive examples

MAY:
- Compare with built-in entity detection
- Review training job CloudTrail events

## Phase 2 — Remediate

MUST:
- Fix training data format (annotations or entity list)
- Provide sufficient annotated examples per entity type
- Create endpoint before inference

SHOULD:
- Balance entity type distribution in training data
- Use test dataset for evaluation
- Monitor endpoint utilization

MAY:
- Implement training data quality validation

## Common Issues

- symptoms: "Recognizer training fails"
  diagnosis: "Training data format incorrect or insufficient examples."
  resolution: "Fix annotation format. Add more examples per entity type."

- symptoms: "Custom entities not detected at inference"
  diagnosis: "Endpoint not created or model accuracy too low."
  resolution: "Create endpoint. Check model metrics. Retrain with more data."

## Output Format

```yaml
root_cause: "custom_entities — <specific_cause>"
evidence:
  - type: recognizer_status
    content: "<recognizer details and metrics>"
  - type: endpoint_status
    content: "<endpoint status>"
severity: HIGH
mitigation:
  immediate: "Fix training data or create endpoint"
  long_term: "Implement entity recognizer lifecycle management"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Training data contains domain-specific labeled examples that may include sensitive content |
| Blast Radius | MEDIUM — Endpoint creation incurs costs; incorrect models affect downstream processing |
| Rollback Complexity | MEDIUM — Endpoints must be deleted; retraining requires time and data |
| Compliance Scope | HIGH — Custom models trained on sensitive data must comply with data retention policies |

## Escalation Conditions

- Escalate to security team if training data contains PII/PHI and S3 bucket lacks encryption or access controls
- Escalate to management if endpoint costs exceed budget thresholds (endpoints bill continuously while active)
- Escalate to data governance if training data retention exceeds policy limits
- Escalate to AWS Support if recognizer training repeatedly fails with no clear error

## Data Sensitivity

- **Training data** in S3 may contain sensitive labeled examples — enforce bucket encryption and access policies
- **Model artifacts** encode patterns from training data — treat as sensitive if trained on sensitive data
- **Endpoint inference results** may reveal sensitive entity patterns
- **IAM data access role** has S3 read access — scope permissions to specific buckets only

## PROHIBITED Actions

- Do NOT leave unused endpoints running — they incur continuous charges
- Do NOT store training data in unencrypted S3 buckets
- Do NOT grant the data access role broader S3 permissions than required
- Do NOT delete training data that may be needed for compliance audits without approval
- Do NOT deploy custom recognizer endpoints to production without validating precision/recall metrics

## Phase 3 — Rollback

- Delete failing endpoint: `aws comprehend delete-endpoint --endpoint-arn <arn>`
- Revert to previous recognizer version by creating a new endpoint pointing to the prior model ARN
- If training data was corrupted, restore from S3 versioned backup and retrain
- Document rollback reason and update model version tracking

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
