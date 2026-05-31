---
title: "H3 — Security Profile Issues"
description: "Diagnose issues with Connect security profiles affecting user access"
status: active
severity: MEDIUM
triggers:
  - "User cannot access feature"
  - "Security profile error"
  - "Missing permissions in CCP"
  - "Admin features not available"
owner: devops-agent
objective: "Fix security profile configuration for Connect users"
context: "Security profiles in Connect control what features users can access within the Connect console and CCP. They are separate from IAM policies. A user must have the correct security profile to access specific features like real-time metrics, recording playback, or user management."
---

## Phase 1 — Triage

MUST:
- Check the user's assigned security profile
- Verify the security profile includes the required permissions
- Compare with a working user's security profile
- Check if the feature requires a specific security profile permission

SHOULD:
- Review the full list of permissions in the security profile
- Check if the user has multiple security profiles (permissions are additive)
- Verify the security profile hasn't been recently modified

MAY:
- Check CloudTrail for security profile changes
- Review Connect documentation for required permissions per feature

## Phase 2 — Remediate

MUST:
- Add the required permissions to the security profile
- Or assign the user to a security profile that has the required permissions

SHOULD:
- Test the user's access after changes
- Document security profile permission requirements
- Review all security profiles for consistency

MAY:
- Create role-based security profiles (agent, supervisor, admin)
- Implement security profile auditing

## Common Issues

- symptoms: "Agent cannot see real-time metrics"
  diagnosis: "Security profile doesn't include metrics access permission."
  resolution: "Add 'Access metrics' permission to the agent's security profile."

- symptoms: "Supervisor cannot listen to recordings"
  diagnosis: "Security profile missing recording playback permission."
  resolution: "Add recording access permissions to the supervisor security profile."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If agent routing or security profiles were modified, restore original profile assignments
2. If recording or streaming configuration was changed, restore original storage settings
3. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "security_profile — <missing permission>"
evidence:
  - type: security_profile
    content: "<profile permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix security profile permissions"
  long_term: "Document and standardize security profiles by role"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
