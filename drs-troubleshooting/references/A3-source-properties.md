---
title: "A3 — Source Server Properties"
description: "Diagnose source server property detection and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "wrong source properties"
  - "disk not detected"
  - "source server info incorrect"
owner: devops-agent
objective: "Resolve source server property detection issues"
context: "DRS collects source server properties (CPU, RAM, disks, OS, network) via the replication agent. Incorrect properties affect launch configuration and recovery instance sizing."
---

## Phase 1 — Triage

MUST:
- Check source properties: `aws drs describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].sourceProperties'`
- Verify disk count and sizes match actual source server
- Check OS and CPU/RAM detection accuracy

SHOULD:
- Compare detected properties with actual server specs
- Verify all data disks are included in replication

MAY:
- Check if dynamic disks or LVM volumes are properly detected

## Phase 2 — Remediate

MUST:
- If disks are missing, verify they were attached before agent installation
- Restart agent if properties are stale

SHOULD:
- Exclude unnecessary disks from replication to reduce costs and sync time
- Update launch configuration to match corrected properties

MAY:
- Use disk-specific replication settings for different performance tiers

## Common Issues

- symptoms: "Source server shows fewer disks than expected"
  diagnosis: "Disks attached after agent installation not detected."
  resolution: "Restart the replication agent to re-detect disk configuration."

## Output Format

```yaml
root_cause: "source_properties — <specific_cause>"
evidence:
  - type: source_properties
    content: "<detected vs actual properties>"
severity: MEDIUM
mitigation:
  immediate: "Correct property detection"
  long_term: "Validate source properties as part of DR readiness checks"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Source properties reveal server configuration details |
| Blast Radius | LOW — Property detection issues affect launch configuration accuracy |
| Rollback Complexity | LOW — Agent restart re-detects properties |
| Compliance Scope | MEDIUM — Accurate properties ensure correct recovery instance sizing |

## Escalation Conditions

- Escalate to operations if missing disks affect replication completeness
- Escalate to DR coordinator if incorrect properties affect recovery instance sizing

## Data Sensitivity

- **Source server properties** (CPU, RAM, disks, OS) reveal infrastructure details
- **Disk configuration** reveals storage architecture

## PROHIBITED Actions

- Do NOT exclude critical data disks from replication
- Do NOT ignore property detection discrepancies — they affect recovery accuracy

## Phase 3 — Rollback

- Restart the agent to re-detect properties
- Update launch configuration to match corrected properties
- Validate with a drill test after property corrections

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
