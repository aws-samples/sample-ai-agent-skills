---
title: "D1 — Parameter Group Issues"
description: "Diagnose RDS Oracle parameter group configuration and application problems"
status: active
severity: MEDIUM
triggers:
  - "parameter group"
  - "pending-reboot"
  - "parameter not applied"
  - "default parameter"
  - "modify parameter"
owner: devops-agent
objective: "Resolve parameter group configuration, application, and compatibility issues"
context: "RDS Oracle uses parameter groups to manage Oracle init parameters. Parameter group families must match the engine version (e.g., oracle-ee-19). Some parameters are static (require reboot), others are dynamic (apply immediately). The default parameter group cannot be modified — create a custom one."
---

## Phase 1 — Triage

MUST:
- Check current parameter group: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBParameterGroups'`
- Check for pending reboot: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBParameterGroups[0].ParameterApplyStatus'`
- List modified parameters: `aws rds describe-db-parameters --db-parameter-group-name <param-group> --source user`
- Check specific parameter: `aws rds describe-db-parameters --db-parameter-group-name <param-group> --query "Parameters[?ParameterName=='<param>']"`

SHOULD:
- Verify parameter group family matches engine: `aws rds describe-db-parameter-groups --db-parameter-group-name <param-group> --query 'DBParameterGroups[0].DBParameterGroupFamily'`
- Check if using default parameter group (cannot be modified)
- Compare with Oracle-side values:
  ```sql
  SELECT name, value, isdefault, ismodified FROM v$parameter
  WHERE ismodified != 'FALSE' ORDER BY name;
  ```

MAY:
- Export all parameter settings for comparison: `aws rds describe-db-parameters --db-parameter-group-name <param-group> --output json > params.json`
- Check parameter group events: `aws rds describe-events --source-type db-parameter-group --duration 1440`

## Phase 2 — Remediate

MUST:
- For default parameter group: create a custom parameter group and associate it
  ```
  aws rds create-db-parameter-group \
    --db-parameter-group-name <custom-name> \
    --db-parameter-group-family oracle-ee-19 \
    --description "Custom Oracle EE 19 parameters"
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --db-parameter-group-name <custom-name> \
    --apply-immediately
  ```
- For pending-reboot parameters: reboot the instance
  ```
  aws rds reboot-db-instance --db-instance-identifier <instance-id>
  ```
- For family mismatch: create parameter group with correct family

SHOULD:
- Modify parameters:
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=<name>,ParameterValue=<value>,ApplyMethod=immediate"
  ```
- Use `ApplyMethod=pending-reboot` for static parameters
- Document all parameter changes for audit

MAY:
- Create parameter group from existing one for testing
- Use AWS CloudFormation for parameter group management

## Common Issues

- symptoms: "Parameter change not taking effect"
  diagnosis: "Static parameter requires reboot. Check ParameterApplyStatus for 'pending-reboot'."
  resolution: "Reboot the instance to apply static parameter changes."

- symptoms: "Cannot modify parameter — using default parameter group"
  diagnosis: "Default parameter groups are read-only."
  resolution: "Create a custom parameter group, modify it, then associate with the instance."

- symptoms: "Parameter group family mismatch after engine upgrade"
  diagnosis: "Old parameter group family (e.g., oracle-ee-12.1) incompatible with new engine (19)."
  resolution: "Create new parameter group with correct family and migrate settings."

## Safety Ratings
- GREEN: describe-db-instances, describe-db-parameters, describe-db-parameter-groups, SELECT from v$parameter — read-only inspection
- YELLOW: modify-db-parameter-group, create-db-parameter-group — recoverable but may require reboot
- RED: reset-db-parameter-group, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Parameter group family mismatch after engine upgrade"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group values (may contain security-sensitive settings)
- MEDIUM: parameter group configuration, apply status, engine version details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix parameter issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest resetting parameter groups to defaults in production without review"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "Document previous parameter values before making changes to enable rollback"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "parameter_group — <specific_cause>"
evidence:
  - type: param_group
    content: "<parameter group name and family>"
  - type: apply_status
    content: "<pending-reboot or in-sync>"
  - type: modified_params
    content: "<user-modified parameters>"
severity: MEDIUM
mitigation:
  immediate: "Apply parameter change or reboot for static params"
  long_term: "Use custom parameter groups with infrastructure-as-code"
```

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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
