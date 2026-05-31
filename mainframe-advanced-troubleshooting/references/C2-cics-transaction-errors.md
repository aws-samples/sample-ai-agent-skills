---
title: "C2 — CICS Transaction Errors"
description: "Diagnose CICS transaction processing errors in Micro Focus environments"
status: active
severity: HIGH
triggers: ["CICS error", "transaction abend", "CICS region not starting", "transaction timeout", "program not found"]
owner: devops-agent
objective: "Resolve CICS transaction processing errors in Micro Focus M2 environments"
context: "CICS is only available in Micro Focus environments. Transaction errors can stem from region configuration, program definitions, resource definitions, or runtime issues."
---

## Phase 1 — Triage

MUST:
- Check environment status (must be Micro Focus): `aws m2 get-environment --environment-id <env-id> --query '{EngineType:engineType,Status:status}'`
- Review CICS-related logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "CICS" --start-time <epoch-ms>`
- Check application status: `aws m2 get-application --application-id <app-id>`

SHOULD:
- Verify CICS region configuration in the application definition
- Check program definitions (PPT entries) for the failing transaction
- Verify resource definitions (FCT, TCT) are correct

MAY:
- Compare CICS configuration with source mainframe settings
- Review transaction routing configuration

## Phase 2 — Remediate

MUST:
- Ensure CICS region is properly configured in the Micro Focus environment
- Verify program definitions exist for all transaction programs
- Fix resource definitions (file control, terminal control) as needed

SHOULD:
- Test transactions individually to isolate failures
- Verify dataset access from CICS programs

MAY:
- Implement CICS transaction monitoring
- Create automated CICS health checks

## Common Issues

- symptoms: "Transaction ABEND with AICA (runaway task)"
  diagnosis: "Transaction exceeded the CICS runaway task interval."
  resolution: "Check for infinite loops in the program. Increase runaway interval if the transaction legitimately requires more time. Review program logic."

## Safety Ratings

```
safety_ratings:
  - "get-environment: GREEN — read-only environment check"
  - "logs filter-log-events: GREEN — read-only log review"
  - "CICS region configuration changes: YELLOW — affects transaction processing"
  - "Program definition changes: YELLOW — affects which programs can execute"
```

## Escalation Conditions

- CICS region fails to start after configuration changes
- Transaction abends affecting critical business processes
- CICS configuration differences between mainframe and M2 cannot be resolved
- Performance degradation in CICS transaction processing

## Data Sensitivity

- CICS region configuration: MEDIUM — reveals transaction processing architecture
- Transaction logs: MEDIUM — may contain business data
- Program definitions: LOW — reveals program names and attributes
- Resource definitions: MEDIUM — reveals data access patterns

## Prohibited Actions

- NEVER modify CICS region configuration in production without a backup
- NEVER attempt CICS troubleshooting in a Blu Age environment (CICS is Micro Focus only)
- NEVER disable CICS security without explicit approval
- NEVER delete program definitions without verifying they are not referenced by active transactions

## Phase 3 — Rollback

1. Restore CICS region configuration from backup
2. Redeploy previous application version if program changes caused issues
3. Verify all transactions are processing correctly after rollback
4. Check CICS region status and resource availability
5. Document the CICS error and resolution

## Output Format

```yaml
root_cause: "cics_transaction — <specific_cause>"
evidence:
  - type: transaction_error
    content: "<CICS abend code and transaction details>"
severity: HIGH
mitigation:
  immediate: "Fix CICS configuration or program issue"
  long_term: "Implement CICS monitoring and automated health checks"
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
