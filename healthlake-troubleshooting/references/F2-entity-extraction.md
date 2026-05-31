---
title: "F2 — Entity Extraction Issues"
description: "Diagnose medical entity extraction quality problems"
status: active
severity: LOW
triggers: ["wrong entities", "missing entities", "low confidence"]
owner: devops-agent
objective: "Identify and resolve entity extraction quality issues"
context: "Extraction issues occur when: text quality poor, medical terminology non-standard, or confidence threshold too high."
---

## Phase 1 — Triage

MUST:
- Check extracted entities in FHIR resource extensions
- Verify source text quality and format
- Check entity confidence scores
- Verify entity categories (MEDICATION, CONDITION, PROCEDURE, etc.)

SHOULD:
- Compare extracted entities with expected entities
- Check if medical abbreviations are being recognized
- Verify language is English (Comprehend Medical primary language)

## Phase 2 — Remediate

MUST:
- Improve source text quality if possible
- Use standard medical terminology
- Filter entities by confidence score in downstream processing

SHOULD:
- Implement post-processing for entity validation
- Document known extraction limitations

## Output Format

```yaml
root_cause: "entity_extraction — <specific_cause>"
evidence:
  - type: extraction_results
    content: "<entities and confidence>"
severity: LOW
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete FHIR datastores containing patient health data without authorization
- Never modify encryption settings on active datastores without compliance review
- Never import untested or unvalidated FHIR data into production datastores
- Never disable access controls or audit logging on datastores with PHI
- Never export patient data without verifying HIPAA compliance requirements

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous datastore configuration from documented state if imports fail
- Revert access control changes immediately if unauthorized access is detected
- Re-enable encryption and audit settings if they were inadvertently modified

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
