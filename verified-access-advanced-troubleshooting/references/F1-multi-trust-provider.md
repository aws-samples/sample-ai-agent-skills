---
title: "F1 — Multi-Trust Provider"
description: "Diagnose multi-trust-provider configuration and evaluation issues"
status: active
severity: HIGH
triggers: ["multiple trust providers", "combined trust evaluation", "user and device trust together", "trust provider conflict"]
owner: devops-agent
objective: "Resolve issues with multiple trust providers attached to a Verified Access instance"
context: "Verified Access instances can have multiple trust providers (e.g., one user-trust and one device-trust). Cedar policies can reference context from all providers. Missing or conflicting provider data causes evaluation failures."
---

## Phase 1 — Triage

MUST:
- List all trust providers on instance: `aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].{Id:VerifiedAccessInstanceId,TrustProviders:VerifiedAccessTrustProviders}'`
- Get each trust provider's details: `aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[*].{Id:VerifiedAccessTrustProviderId,Type:TrustProviderType,PolicyRef:PolicyReferenceName}'`
- Check access logs for context from each provider: `aws logs filter-log-events --log-group-name <va-log-group> --start-time <epoch-ms> --limit 20`

SHOULD:
- Verify each provider's policy_reference_name is unique
- Check Cedar policy references all required provider contexts
- Verify each provider is returning data independently

MAY:
- Test with each trust provider individually to isolate issues
- Review Cedar policy logic for multi-provider conditions

## Phase 2 — Remediate

MUST:
- Ensure each trust provider has a unique policy_reference_name
- Update Cedar policies to correctly reference each provider's context path
- Verify all trust providers are operational and returning data

SHOULD:
- Test Cedar policies with both user and device context present
- Handle cases where one provider's context may be temporarily unavailable

MAY:
- Implement fallback logic in Cedar policies for optional trust providers
- Create monitoring for individual trust provider health

## Common Issues

- symptoms: "Access denied when both user and device trust are required but only one context is present"
  diagnosis: "One trust provider is not returning context data, causing Cedar policy to fail."
  resolution: "Check each trust provider independently. Verify agent enrollment for device trust. Check OIDC configuration for user trust. Update Cedar policy to handle missing context gracefully."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-instances: GREEN — read-only enumeration"
  - "describe-verified-access-trust-providers: GREEN — read-only enumeration"
  - "Access log analysis: GREEN — read-only log review"
  - "Detach trust provider: RED — removes trust evaluation capability"
```

## Escalation Conditions

- Multiple trust providers returning conflicting data
- One trust provider consistently unavailable affecting all access decisions
- Cedar policy complexity exceeds maintainability with multiple providers
- Trust provider vendor integration issues requiring vendor support

## Data Sensitivity

- Trust provider configurations: MEDIUM — reveals authentication architecture
- Combined trust context: HIGH — contains user identity and device posture
- Access logs with multi-provider context: HIGH — contains comprehensive access data
- Cedar policies with multi-provider logic: MEDIUM — reveals authorization architecture

## Prohibited Actions

- NEVER detach a trust provider from a production instance without understanding the impact on Cedar policies
- NEVER assume one trust provider can substitute for another
- NEVER create Cedar policies that ignore device trust when it's required by security policy
- NEVER modify trust provider configurations without testing the impact on multi-provider evaluation

## Phase 3 — Rollback

1. If a trust provider was detached, re-attach it: `aws ec2 attach-verified-access-trust-provider --verified-access-instance-id <id> --verified-access-trust-provider-id <tp-id>`
2. Revert Cedar policy to previous version that worked with all providers
3. Verify each trust provider is returning context data
4. Test access with users who require both user and device trust
5. Document the multi-provider configuration and dependencies

## Output Format

```yaml
root_cause: "multi_trust_provider — <specific_cause>"
evidence:
  - type: trust_evaluation
    content: "<provider configurations and context data>"
severity: HIGH
mitigation:
  immediate: "Fix trust provider or Cedar policy issue"
  long_term: "Implement per-provider monitoring and policy testing"
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
