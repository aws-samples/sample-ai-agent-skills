---
title: "A2 — Trust Provider Advanced"
description: "Diagnose advanced trust provider configuration issues"
status: active
severity: HIGH
triggers: ["trust provider not working", "OIDC configuration error", "device trust provider failed", "trust context missing"]
owner: devops-agent
objective: "Resolve advanced trust provider configuration and integration issues"
context: "Trust providers supply identity and device context to Cedar policies. User-trust providers use OIDC or IAM Identity Center. Device-trust providers integrate with CrowdStrike, Jamf, etc."
---

## Phase 1 — Triage

MUST:
- List trust providers: `aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[*].{Id:VerifiedAccessTrustProviderId,Type:TrustProviderType,UserTrust:UserTrustProviderType,DeviceTrust:DeviceTrustProviderType,PolicyRef:PolicyReferenceName}'`
- Verify trust provider is attached to instance: `aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].{Id:VerifiedAccessInstanceId,TrustProviders:VerifiedAccessTrustProviders}'`
- Check access logs for trust context issues: `aws logs filter-log-events --log-group-name <va-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms>`

SHOULD:
- Verify OIDC provider configuration (issuer URL, client ID, scopes)
- Check device trust provider enrollment status
- Verify policy_reference_name matches Cedar policy references

MAY:
- Test OIDC token exchange manually
- Review device trust provider vendor dashboard for agent status

## Phase 2 — Remediate

MUST:
- Ensure OIDC issuer URL is reachable and returns valid discovery document
- Verify client ID and client secret are correct for OIDC providers
- Confirm policy_reference_name in trust provider matches Cedar policy context references

SHOULD:
- Test OIDC authentication flow end-to-end
- Verify device trust agent is installed and enrolled on test devices

MAY:
- Configure additional OIDC scopes for richer trust context
- Set up monitoring for trust provider availability

## Common Issues

- symptoms: "Cedar policy denies access despite correct user identity"
  diagnosis: "policy_reference_name mismatch between trust provider and Cedar policy."
  resolution: "Verify policy_reference_name: aws ec2 describe-verified-access-trust-providers. Update Cedar policy to use the correct reference name."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-trust-providers: GREEN — read-only enumeration"
  - "ModifyVerifiedAccessTrustProvider: YELLOW — changes trust configuration"
  - "OIDC token testing: YELLOW — involves authentication credentials"
  - "Access log analysis: GREEN — read-only log review"
```

## Escalation Conditions

- OIDC provider is unreachable or returning invalid tokens
- Device trust provider vendor reports integration issues
- Trust context data is incomplete or incorrect despite proper configuration
- Multiple trust providers conflict in policy evaluation

## Data Sensitivity

- `aws ec2 describe-verified-access-trust-providers`: MEDIUM — reveals trust provider configuration
- OIDC client credentials: HIGH — authentication secrets
- Access logs with trust context: HIGH — contains user identity and device information
- Device trust data: HIGH — reveals device compliance and security posture

## Prohibited Actions

- NEVER expose OIDC client secrets in logs, documentation, or error messages
- NEVER delete a trust provider attached to an active Verified Access instance
- NEVER modify trust provider configuration during peak access hours without a rollback plan
- NEVER bypass trust evaluation by creating overly permissive Cedar policies

## Phase 3 — Rollback

1. If trust provider configuration was modified, revert to previous settings: `aws ec2 modify-verified-access-trust-provider --verified-access-trust-provider-id <id> --oidc-options Issuer=<original>,ClientId=<original>`
2. If a trust provider was detached, re-attach it: `aws ec2 attach-verified-access-trust-provider --verified-access-instance-id <inst-id> --verified-access-trust-provider-id <tp-id>`
3. Verify Cedar policies still reference the correct policy_reference_name
4. Test access with a known-good user/device combination
5. Document configuration changes and their effects

## Output Format

```yaml
root_cause: "trust_provider — <specific_cause>"
evidence:
  - type: trust_config
    content: "<trust provider details and context data>"
severity: HIGH
mitigation:
  immediate: "Fix trust provider configuration"
  long_term: "Implement trust provider monitoring and testing"
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
