---
title: "D2 — Camera Stream Failures"
description: "Diagnose runtime camera stream disconnections and quality issues"
status: active
severity: MEDIUM
triggers:
  - "camera stream dropped"
  - "video feed intermittent"
  - "frame rate low"
owner: devops-agent
objective: "Resolve runtime camera stream failures"
context: "Camera streams can drop due to network issues, camera reboots, or bandwidth constraints."
---

## Phase 1 — Triage

MUST:
- Check application logs for stream disconnect events
- Verify camera is still powered and connected
- Check network bandwidth between camera and appliance
- Monitor frame rate in application metrics

SHOULD:
- Check for network congestion or packet loss
- Verify camera is not being accessed by too many clients
- Check camera health/status page

MAY:
- Reduce stream resolution to lower bandwidth requirements
- Check for camera firmware issues

## Phase 2 — Remediate

MUST:
- Restore network connectivity if disrupted
- Restart camera if unresponsive
- Redeploy application if stream cannot reconnect

SHOULD:
- Configure stream reconnection logic in application code
- Use lower resolution streams for bandwidth-constrained networks

## Common Issues

- symptoms: "Intermittent frame drops"
  diagnosis: "Network bandwidth insufficient for stream resolution."
  resolution: "Reduce camera stream resolution or improve network bandwidth."

## Output Format

```yaml
root_cause: "stream_failure — <specific_cause>"
evidence:
  - type: stream_health
    content: "<stream connectivity and quality>"
severity: MEDIUM
mitigation:
  immediate: "Restore stream connectivity"
  long_term: "Implement stream monitoring and auto-recovery"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — stream failures may involve sensitive video content
- Service Disruption Risk: HIGH — stream failures create monitoring blind spots
- Compliance Impact: HIGH — monitoring gaps may violate security requirements

## Escalation Conditions
- All streams fail simultaneously indicating infrastructure issue
- Stream failures create security monitoring gaps
- Stream recovery does not occur automatically
- Stream failures correlate with network changes
- Persistent stream failures on critical cameras

## Data Sensitivity
- Stream error logs — INTERNAL
- Camera health data — INTERNAL
- Network diagnostics — INTERNAL
- Video content references — RESTRICTED
- Never expose video content in troubleshooting logs

## Prohibited Actions
- Do NOT ignore stream failures on security-critical cameras
- Do NOT bypass stream authentication for troubleshooting
- Do NOT expose stream diagnostic data externally
- Do NOT disable stream monitoring during active failures
- Do NOT modify stream configurations without testing
- Do NOT accept degraded stream quality without investigation

## Phase 3 — Rollback
- Restore previous stream configurations
- Re-establish failed camera connections
- Revert network changes that caused stream failures
- Verify all streams are active after recovery
- Restore monitoring alerts for stream health
- Document stream failure patterns and resolutions

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
