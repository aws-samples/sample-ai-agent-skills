---
title: "B2 — Provider Service Errors"
description: "Diagnose third-party provider service errors in Entity Resolution including subscription, connectivity, and configuration issues"
status: active
severity: HIGH
triggers: ["provider error", "provider service failed", "LiveRamp error", "TransUnion error", "third-party service issue"]
owner: devops-agent
objective: "Resolve provider service integration errors"
context: "Provider services (LiveRamp, TransUnion, etc.) require separate subscriptions and have independent error modes. Entity Resolution provides the integration framework, but provider issues are often subscription or configuration related."
---

## Phase 1 — Triage

MUST:
- List available provider services: `aws entityresolution list-provider-services`
- Get provider service details: `aws entityresolution get-provider-service --provider-name <name> --provider-service-name <service>`
- Check the workflow configuration for provider settings

SHOULD:
- Verify provider subscription status (may require provider portal)
- Check the service role has permissions for provider-related resources
- Review job error messages for provider-specific error codes

MAY:
- Contact provider support for service-specific issues
- Check provider service status page for outages
- Review CloudTrail for provider-related API calls

## Phase 2 — Remediate

MUST:
- Verify and renew provider subscription if expired
- Fix provider configuration in the workflow
- Ensure service role has required permissions for provider integration

SHOULD:
- Test provider connectivity with a small dataset
- Document provider-specific configuration requirements
- Set up monitoring for provider service availability

MAY:
- Implement fallback matching strategies when provider is unavailable
- Configure retry logic for transient provider errors

## Safety Ratings

```
safety_ratings:
  - "list-provider-services: GREEN — read-only provider listing"
  - "get-provider-service: GREEN — read-only provider details"
  - "Workflow with provider: YELLOW — sends data to third-party service"
  - "Provider subscription management: YELLOW — affects service access"
```

## Escalation Conditions

- Provider service outage affects business-critical matching
- Provider subscription renewal requires procurement approval
- Provider-specific errors not documented in Entity Resolution docs
- Data sharing with provider raises compliance concerns

## Data Sensitivity

- `aws entityresolution list-provider-services`: LOW — returns provider names
- `aws entityresolution get-provider-service`: LOW — returns provider configuration
- Data sent to provider: HIGH — contains records for matching/mapping
- Provider credentials: HIGH — must be protected

## Prohibited Actions

- NEVER share provider API keys or credentials in logs or documentation
- NEVER send production PII to provider services without data processing agreements
- NEVER bypass provider rate limits by creating multiple workflows
- NEVER use provider services without verifying compliance with data regulations

## Phase 3 — Rollback

1. If provider configuration was changed, restore previous settings
2. If a workflow was modified for provider troubleshooting, revert changes
3. If provider subscription was modified, verify service access
4. Notify downstream systems if provider results are affected
5. Document the provider issue and resolution

## Common Issues

- symptoms: "Workflow fails with provider authentication error"
  diagnosis: "Provider subscription is expired or credentials are invalid."
  resolution: "Verify provider subscription status. Update credentials if expired. Contact provider support."

- symptoms: "Provider returns partial results or empty response"
  diagnosis: "Input data format does not match provider requirements, or provider rate limits are exceeded."
  resolution: "Check provider documentation for input format requirements. Reduce batch size if hitting rate limits."

- symptoms: "Provider service not found in list-provider-services"
  diagnosis: "The provider service is not available in the current region, or the provider name is incorrect."
  resolution: "Check regional availability. Verify exact provider and service names from documentation."

## Output Format

```yaml
root_cause: "provider_service — <specific_cause>"
evidence:
  - type: provider_error
    content: "<provider error details>"
severity: HIGH
mitigation:
  immediate: "Fix provider configuration"
  long_term: "Implement provider health monitoring and fallback"
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
