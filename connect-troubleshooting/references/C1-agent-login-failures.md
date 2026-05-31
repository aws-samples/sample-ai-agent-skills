---
title: "C1 — Agent Login Failures"
description: "Diagnose failures when agents cannot log in to the CCP or Connect"
status: active
severity: HIGH
triggers:
  - "Agent cannot log in"
  - "CCP login error"
  - "Access denied"
  - "SSO login failed"
owner: devops-agent
objective: "Restore agent login access"
context: "Agent login can fail due to: user not created in Connect, SAML/SSO misconfiguration, security profile missing, browser issues, network/firewall blocking, or instance access URL incorrect."
---

## Phase 1 — Triage

MUST:
- Verify the user exists in the Connect instance: `aws connect describe-user --instance-id <id> --user-id <user-id>`
- Check the login method (Connect-managed, SAML, or IAM Identity Center)
- Verify the instance access URL is correct
- Check if the issue affects all agents or specific ones

SHOULD:
- Check SAML/SSO configuration if using federated login
- Verify the agent's security profile allows CCP access
- Check browser compatibility (Chrome or Firefox required)
- Check network connectivity to Connect domains

MAY:
- Check CloudTrail for authentication events
- Review browser console for JavaScript errors

## Phase 2 — Remediate

MUST:
- Create the user in Connect if missing
- Fix SAML/SSO configuration if federated login is failing
- Ensure the correct instance URL is being used

SHOULD:
- Verify security profile permissions
- Test with a different browser
- Check VPN/firewall rules for Connect domain access

MAY:
- Implement SSO troubleshooting guide for agents
- Set up login monitoring and alerting

## Common Issues

- symptoms: "Agent gets 'Access Denied' after SSO login"
  diagnosis: "SAML assertion doesn't map to a Connect user, or user doesn't exist."
  resolution: "Verify SAML attribute mapping matches Connect username. Create the user if missing."

- symptoms: "Login page loads but CCP doesn't initialize"
  diagnosis: "Browser or network issue blocking WebRTC or Connect domains."
  resolution: "Check browser compatibility. Whitelist Connect domains and ports."

- symptoms: "Some agents can log in, others cannot"
  diagnosis: "Affected agents missing from Connect user directory or have incorrect security profiles."
  resolution: "Verify user exists and has appropriate security profile."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
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
2. If phone number associations were changed, reassociate the number with the original contact flow
3. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "agent_login — <specific cause>"
evidence:
  - type: user_config
    content: "<user and authentication details>"
severity: HIGH
mitigation:
  immediate: "Fix the login issue for affected agents"
  long_term: "Implement login monitoring and SSO validation"
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
