---
title: "D2 — Device Trust Provider Issues"
description: "Diagnose device trust provider integration problems"
status: active
severity: MEDIUM
triggers:
  - "device trust error"
  - "device posture"
  - "CrowdStrike integration"
  - "Jamf integration"
owner: devops-agent
objective: "Identify and resolve device trust provider issues"
context: "Device trust providers supply device posture context (compliance status, OS version, etc.) for Cedar policy evaluation. Supported providers include CrowdStrike, Jamf, and others. Device context is optional but enables policies that consider device security posture."
---

## Phase 1 — Triage

MUST:
- Check device trust provider config: `aws ec2 describe-verified-access-trust-providers --verified-access-trust-provider-ids <provider-id>`
- Verify provider is attached to the Verified Access instance
- Check if device is enrolled with the device trust provider
- Verify policy reference name is correct in Cedar policies

SHOULD:
- Check device provider admin console for device enrollment status
- Verify device posture data is being reported
- Check access logs for device context availability

MAY:
- Test with a known-compliant device
- Check device provider API connectivity

## Phase 2 — Remediate

MUST:
- Ensure device is enrolled with the trust provider
- Verify trust provider configuration matches provider's API settings
- Use correct context path in Cedar: `context.<device-policy-ref>.<attribute>`

SHOULD:
- Test device context availability before enforcing device policies
- Start with permissive device policies and tighten gradually

MAY:
- Configure device compliance requirements in the provider
- Implement fallback policies for devices without trust data

## Common Issues

- symptoms: "Device context missing in policy evaluation"
  diagnosis: "Device not enrolled or provider not properly configured."
  resolution: "Enroll device with trust provider. Verify provider configuration."

- symptoms: "Device trust provider attachment fails"
  diagnosis: "Instance already has a device trust provider."
  resolution: "Only one device provider per instance. Detach existing before attaching new."

- symptoms: "Device shows non-compliant"
  diagnosis: "Device doesn't meet compliance requirements in provider."
  resolution: "Check device compliance in provider admin console. Remediate device."

## Output Format

```yaml
root_cause: "device_trust_provider — <specific_cause>"
evidence:
  - type: provider_config
    content: "<device trust provider configuration>"
  - type: device_enrollment
    content: "<device enrollment status>"
  - type: device_context
    content: "<device posture data>"
severity: MEDIUM
mitigation:
  immediate: "Fix device enrollment or provider configuration"
  long_term: "Implement device compliance monitoring"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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
