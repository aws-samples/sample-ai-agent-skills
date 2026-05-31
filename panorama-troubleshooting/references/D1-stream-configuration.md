---
title: "D1 — Camera Stream Configuration"
description: "Diagnose camera stream setup and RTSP configuration issues"
status: active
severity: MEDIUM
triggers:
  - "camera not connecting"
  - "RTSP stream error"
  - "no video feed"
owner: devops-agent
objective: "Resolve camera stream configuration issues"
context: "Panorama connects to cameras via RTSP. Cameras must be network-reachable from the appliance with correct URI and credentials."
---

## Phase 1 — Triage

MUST:
- Verify camera RTSP URI format: `rtsp://user:pass@ip:port/path`
- Confirm camera is powered on and network-connected
- Verify camera is reachable from the Panorama appliance subnet
- Check application logs for stream connection errors

SHOULD:
- Test RTSP stream with a tool like VLC from the same network
- Verify camera credentials are correct
- Check for firewall rules blocking RTSP port (typically 554)

MAY:
- Verify camera firmware supports RTSP
- Check for camera-specific RTSP path format

## Phase 2 — Remediate

MUST:
- Correct RTSP URI format and credentials
- Ensure network connectivity between appliance and camera
- Open RTSP port in any firewalls

SHOULD:
- Document camera RTSP URIs and credentials
- Test streams before deploying applications

## Common Issues

- symptoms: "Stream timeout in application logs"
  diagnosis: "Camera not reachable or RTSP URI incorrect."
  resolution: "Verify URI format, credentials, and network path. Test with VLC first."

## Output Format

```yaml
root_cause: "stream_config — <specific_cause>"
evidence:
  - type: stream_status
    content: "<camera stream connectivity>"
severity: MEDIUM
mitigation:
  immediate: "Fix RTSP configuration"
  long_term: "Document and validate camera configurations"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — video streams may contain sensitive or private content
- Service Disruption Risk: MEDIUM — stream configuration issues affect monitoring coverage
- Compliance Impact: HIGH — video stream handling must comply with privacy regulations

## Escalation Conditions
- Camera streams fail to connect to Panorama device
- Stream configuration changes cause monitoring gaps
- Stream quality degrades affecting inference accuracy
- Multiple streams fail simultaneously
- Stream credentials expire or become invalid

## Data Sensitivity
- Camera stream URLs and credentials — RESTRICTED
- Video stream content — RESTRICTED
- Stream configuration parameters — INTERNAL
- Camera network details — INTERNAL
- Never expose camera credentials or stream URLs

## Prohibited Actions
- Do NOT transmit video streams over unencrypted connections
- Do NOT store camera credentials in plaintext
- Do NOT configure streams from cameras without proper authorization
- Do NOT expose stream URLs to unauthorized users
- Do NOT bypass video stream privacy controls
- Do NOT retain video data beyond approved retention periods

## Phase 3 — Rollback
- Restore previous stream configurations
- Re-establish camera connections with correct credentials
- Revert stream quality settings
- Verify monitoring coverage after stream changes
- Rotate camera credentials if exposed
- Document stream configuration changes and their impact

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
