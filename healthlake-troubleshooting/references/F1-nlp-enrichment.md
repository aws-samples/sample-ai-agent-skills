---
title: "F1 — NLP Enrichment Issues"
description: "Diagnose Comprehend Medical NLP enrichment problems"
status: active
severity: MEDIUM
triggers: ["NLP not working", "entities not extracted", "enrichment failed"]
owner: devops-agent
objective: "Identify and resolve NLP enrichment issues"
context: "NLP issues occur when: Comprehend Medical not enabled, IAM permissions missing, or clinical notes not in expected format."
---

## Phase 1 — Triage

MUST:
- Verify Comprehend Medical integration is enabled on data store
- Check IAM permissions for comprehendmedical:DetectEntitiesV2
- Verify clinical notes are in text format within FHIR resources
- Check import job for NLP-related errors

SHOULD:
- Verify enrichment extensions are added to imported resources
- Check if specific resource types are configured for enrichment
- Review Comprehend Medical service limits

## Phase 2 — Remediate

MUST:
- Enable Comprehend Medical integration
- Grant IAM permissions for Comprehend Medical
- Ensure clinical notes are in processable text format

SHOULD:
- Test with sample clinical notes
- Monitor enrichment success rates

## Output Format

```yaml
root_cause: "nlp_enrichment — <specific_cause>"
evidence:
  - type: enrichment_status
    content: "<enrichment details>"
severity: MEDIUM
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
