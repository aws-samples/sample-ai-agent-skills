---
title: "D1 — Device Trust Integration"
description: "Diagnose device trust provider integration issues"
status: active
severity: HIGH
triggers: ["device trust not working", "CrowdStrike integration failed", "Jamf not reporting", "device posture missing", "device context empty"]
owner: devops-agent
objective: "Resolve device trust provider integration issues with Verified Access"
context: "Device trust providers (CrowdStrike, Jamf) supply device posture data to Cedar policies. Integration requires provider configuration, agent deployment, and correct Cedar policy references."
---

## Phase 1 — Triage

MUST:
- Check device trust provider configuration: `aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[?TrustProviderType==\`device\`].{Id:VerifiedAccessTrustProviderId,DeviceType:DeviceTrustProviderType,PolicyRef:PolicyReferenceName}'`
- Review access logs for device context: `aws logs filter-log-events --log-group-name <va-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms>`
- Verify trust provider is attached to instance: `aws ec2 describe-verified-access-instances`

SHOULD:
- Verify device trust agent is installed on the end-user device
- Check agent enrollment status with the trust provider vendor
- Confirm Cedar policy references the correct device context path

MAY:
- Test with a known-compliant device to isolate device-specific issues
- Review vendor documentation for integration requirements

## Phase 2 — Remediate

MUST:
- Ensure device trust provider is correctly configured with vendor credentials
- Verify agent is installed, enrolled, and reporting on end-user devices
- Update Cedar policy to reference device context correctly: `context.<policy_ref>.device`

SHOULD:
- Test device trust evaluation with a controlled device
- Document required agent versions and enrollment procedures

MAY:
- Implement device compliance monitoring dashboards
- Set up alerts for device trust evaluation failures

## Common Issues

- symptoms: "Device context is empty in access logs despite agent installed"
  diagnosis: "Agent not enrolled with the trust provider or provider credentials misconfigured."
  resolution: "Verify agent enrollment in vendor dashboard. Check trust provider configuration: aws ec2 describe-verified-access-trust-providers."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-trust-providers: GREEN — read-only enumeration"
  - "Access log analysis: GREEN — read-only log review"
  - "ModifyVerifiedAccessTrustProvider: YELLOW — changes trust configuration"
  - "Agent deployment: YELLOW — installs software on end-user devices"
```

## Escalation Conditions

- Device trust provider vendor reports API or integration issues
- Agent deployment failures across multiple devices
- Device context data is incorrect or stale
- Trust provider configuration requires vendor-side changes

## Data Sensitivity

- Device trust provider configuration: MEDIUM — reveals vendor integration details
- Device posture data: HIGH — reveals device security compliance status
- Access logs with device context: HIGH — contains device identifiers and posture
- Vendor API credentials: HIGH — authentication secrets for trust provider

## Prohibited Actions

- NEVER expose vendor API credentials in logs or documentation
- NEVER disable device trust evaluation to bypass connectivity issues
- NEVER deploy agents to devices without endpoint management approval
- NEVER assume device compliance without verifying actual posture data

## Phase 3 — Rollback

1. If trust provider configuration was modified, revert to previous settings
2. If Cedar policy was changed, restore previous policy version
3. If agent was deployed to incorrect devices, coordinate removal with endpoint management
4. Verify device trust evaluation works for known-compliant devices
5. Document integration changes and their effects

## Output Format

```yaml
root_cause: "device_trust — <specific_cause>"
evidence:
  - type: device_context
    content: "<trust provider config and device posture data>"
severity: HIGH
mitigation:
  immediate: "Fix device trust integration"
  long_term: "Implement device compliance monitoring"
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
