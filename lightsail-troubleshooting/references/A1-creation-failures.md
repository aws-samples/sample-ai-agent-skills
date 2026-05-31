---
title: "A1 — Instance Creation Failures"
description: "Diagnose why a Lightsail instance creation is failing"
status: active
severity: HIGH
triggers:
  - "instance creation failed"
  - "cannot create instance"
  - "Lightsail launch error"
  - "instance not starting"
owner: devops-agent
objective: "Identify why Lightsail instance creation is failing and resolve the issue"
context: "Instance creation failures can result from account limits, invalid blueprint/bundle combinations, availability zone capacity, IAM permission issues, or SSH key problems. Lightsail has per-region instance limits."
---

## Phase 1 — Triage

MUST:
- Check account instance limits: `aws lightsail get-instances --query 'instances | length(@)'`
- Verify blueprint availability: `aws lightsail get-blueprints --query 'blueprints[*].{Id:blueprintId,Name:name,Type:type,Active:isActive}'`
- Verify bundle availability: `aws lightsail get-bundles --query 'bundles[*].{Id:bundleId,Name:name,Price:price,Active:isActive}'`
- Check for recent creation errors in CloudTrail

SHOULD:
- Verify SSH key pair exists: `aws lightsail get-key-pairs`
- Check availability zone capacity
- Verify IAM permissions for Lightsail operations

MAY:
- Check if the region supports the desired blueprint
- Review account-level service quotas

## Phase 2 — Remediate

MUST:
- Use valid blueprint and bundle combination
- Ensure account is within instance limits (20 per region default)
- Create instance: `aws lightsail create-instances --instance-names <name> --availability-zone <az> --blueprint-id <blueprint> --bundle-id <bundle> --key-pair-name <key>`

SHOULD:
- Request limit increase if at capacity
- Use a supported blueprint for the region

MAY:
- Use launch script for automated configuration: `--user-data <script>`
- Create instance from snapshot for faster deployment

## Common Issues

- symptoms: "Instance limit exceeded"
  diagnosis: "Account has reached the 20-instance per-region limit."
  resolution: "Delete unused instances or request a limit increase."

- symptoms: "Invalid blueprint or bundle"
  diagnosis: "Blueprint/bundle combination not available or deprecated."
  resolution: "List available blueprints and bundles. Use active ones only."

- symptoms: "SSH key not found"
  diagnosis: "Specified key pair does not exist in the region."
  resolution: "Create or import SSH key pair: aws lightsail create-key-pair --key-pair-name <name>"

## Output Format

```yaml
root_cause: "creation_failure — <specific_cause>"
evidence:
  - type: instance_count
    content: "<current instance count vs limit>"
  - type: blueprint_bundle
    content: "<requested blueprint and bundle>"
  - type: error_details
    content: "<specific error message>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Monitor instance counts and automate with IaC"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
