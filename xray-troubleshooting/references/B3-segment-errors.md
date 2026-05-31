---
title: "B3 — Segment Errors"
description: "Diagnose X-Ray segment document errors and rejections"
status: active
severity: MEDIUM
triggers:
  - "segment rejected"
  - "segment too large"
  - "segment error"
  - "invalid segment"
owner: devops-agent
objective: "Identify and resolve segment document errors"
context: "Segment documents have a 64KB size limit. Malformed segments, oversized documents, or invalid fields cause rejections. The daemon logs rejection errors. Segment errors can also occur from incorrect SDK usage."
---

## Phase 1 — Triage

MUST:
- Check daemon logs for segment rejection errors
- Verify segment document size is under 64KB
- Check for malformed segment fields (missing required fields)
- Review application logs for X-Ray SDK errors

SHOULD:
- Check if annotations or metadata are excessively large
- Verify subsegment nesting depth is reasonable
- Check for duplicate segment IDs

MAY:
- Enable SDK debug logging to see segment documents before sending
- Check X-Ray API error metrics in CloudWatch

## Phase 2 — Remediate

MUST:
- Reduce segment document size below 64KB
- Fix malformed segment fields
- Remove excessively large annotations or metadata

SHOULD:
- Limit the number of subsegments per segment
- Use metadata instead of annotations for non-searchable data
- Keep annotations to essential key-value pairs only

MAY:
- Implement segment size monitoring
- Create custom subsegment strategies to control document size

## Common Issues

- symptoms: "Segments rejected with size error"
  diagnosis: "Segment document exceeds 64KB limit."
  resolution: "Reduce annotations, metadata, and subsegment count. Keep under 64KB."

- symptoms: "Invalid segment format errors in daemon logs"
  diagnosis: "SDK generating malformed segment documents."
  resolution: "Update SDK to latest version. Check for custom segment creation errors."

- symptoms: "Subsegments not appearing despite SDK instrumentation"
  diagnosis: "Subsegments created outside of an active segment context."
  resolution: "Ensure subsegments are created within an active segment. Check async context propagation."

## Output Format

```yaml
root_cause: "segment_error — <specific_cause>"
evidence:
  - type: daemon_logs
    content: "<segment rejection errors>"
  - type: segment_size
    content: "<estimated segment document size>"
  - type: sdk_errors
    content: "<SDK error messages>"
severity: MEDIUM
mitigation:
  immediate: "Fix segment document issues"
  long_term: "Implement segment size monitoring and limits"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
