---
title: "H1 — SAML/SSO Login Failures"
description: "Diagnose SAML-based SSO authentication failures for Connect"
status: active
severity: HIGH
triggers:
  - "SAML login failed"
  - "SSO error"
  - "Federation error"
  - "Cannot authenticate"
owner: devops-agent
objective: "Restore SAML/SSO authentication for Connect users"
context: "Connect supports SAML 2.0 for federated authentication. The SAML assertion must include specific attributes that map to Connect usernames. Common failures: incorrect assertion, attribute mapping mismatch, certificate expiry, or relay state misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check the SAML error message (usually visible in the browser)
- Verify the SAML IdP metadata is correctly configured in Connect
- Check if the SAML assertion includes the required username attribute
- Verify the relay state URL is correct for the Connect instance

SHOULD:
- Decode the SAML assertion to check attribute values
- Verify the IdP signing certificate hasn't expired
- Check if the user exists in Connect with a matching username
- Verify the SAML assertion audience matches the Connect instance

MAY:
- Check IdP logs for authentication errors
- Test with a SAML debugging tool (browser extension)

## Phase 2 — Remediate

MUST:
- Fix SAML attribute mapping to match Connect username format
- Update expired IdP certificates
- Correct the relay state URL
- Ensure the user exists in Connect

SHOULD:
- Test SAML login with a known-good user
- Document the SAML configuration for future reference
- Verify all required SAML attributes are present

MAY:
- Implement SAML assertion logging for debugging
- Set up certificate expiry monitoring

## Common Issues

- symptoms: "SAML login redirects but returns error"
  diagnosis: "SAML assertion attribute doesn't match Connect username."
  resolution: "Fix the attribute mapping in the IdP to match the Connect username format."

- symptoms: "Certificate error during SAML login"
  diagnosis: "IdP signing certificate expired or not trusted."
  resolution: "Update the certificate in the Connect SAML configuration."

- symptoms: "Login succeeds but user gets access denied"
  diagnosis: "SAML username maps to a user that doesn't exist in Connect."
  resolution: "Create the user in Connect or fix the username mapping."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
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
root_cause: "saml_sso — <specific cause>"
evidence:
  - type: saml_assertion
    content: "<assertion details and error>"
severity: HIGH
mitigation:
  immediate: "Fix SAML configuration"
  long_term: "Implement certificate monitoring and SAML validation"
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
