---
title: "G1 — DNSSEC Validation"
description: "Diagnose DNSSEC validation configuration and resolution failures"
status: active
severity: HIGH
triggers: ["DNSSEC validation failed", "SERVFAIL after DNSSEC", "DNSSEC configuration error"]
owner: devops-agent
objective: "Resolve DNSSEC validation issues causing DNS resolution failures"
context: "DNSSEC validation verifies DNS response authenticity. When enabled per-VPC, queries for domains with invalid DNSSEC signatures return SERVFAIL. This can break resolution for misconfigured domains."
---

## Phase 1 — Triage

MUST:
- Check DNSSEC validation status: `aws route53resolver list-resolver-dnssec-configs --query 'ResolverDnssecConfigs[*].{Id:Id,ResourceId:ResourceId,ValidationStatus:ValidationStatus}'`
- Identify failing domains from query logs or user reports
- Test DNS resolution for the failing domain: `dig +dnssec <domain>`

SHOULD:
- Check if the domain has valid DNSSEC signatures: `dig +dnssec +cd <domain>` (bypass validation)
- Verify the domain's DS records in the parent zone

MAY:
- Use external DNSSEC validation tools (dnsviz.net, verisignlabs.com/dnssec)
- Check if the issue is with the domain's DNSSEC configuration, not the resolver

## Phase 2 — Remediate

MUST:
- If DNSSEC is breaking resolution for critical domains, evaluate disabling: `aws route53resolver update-resolver-dnssec-config --resource-id <vpc-id> --validation DISABLE`
- If the domain's DNSSEC is misconfigured, contact the domain owner
- Document which domains fail DNSSEC validation

SHOULD:
- Test DNSSEC validation in a non-production VPC before enabling in production
- Monitor for SERVFAIL responses after enabling DNSSEC

MAY:
- Set up CloudWatch alarms for DNSSEC validation failures
- Create a whitelist of domains known to have DNSSEC issues

## Common Issues

- symptoms: "DNS resolution returns SERVFAIL for specific domains after enabling DNSSEC"
  diagnosis: "Domain has invalid, expired, or missing DNSSEC signatures."
  resolution: "Contact domain owner to fix DNSSEC. Temporarily disable DNSSEC validation if critical."

- symptoms: "DNSSEC validation enabled but no visible effect"
  diagnosis: "Most domains don't have DNSSEC configured, so validation has no effect on them."
  resolution: "DNSSEC validation only affects domains with DNSSEC records. This is expected behavior."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-dnssec-configs: GREEN — read-only DNSSEC status"
  - "update-resolver-dnssec-config ENABLE: YELLOW — may break resolution for misconfigured domains"
  - "update-resolver-dnssec-config DISABLE: YELLOW — removes DNS authenticity verification"
  - "dig +dnssec: GREEN — read-only DNS query with DNSSEC data"
```

## Escalation Conditions

- DNSSEC validation breaks resolution for critical production services
- Multiple domains fail DNSSEC validation simultaneously
- Domain owner is unresponsive about fixing DNSSEC configuration
- Compliance requires DNSSEC but it breaks required third-party services

## Data Sensitivity

- DNSSEC configuration: LOW — validation enabled/disabled status
- DNS query results: MEDIUM — reveal domain names and resolution data
- DNSSEC signatures: LOW — public cryptographic data
- Failing domain list: MEDIUM — reveals DNS dependencies

## Prohibited Actions

- NEVER enable DNSSEC validation in production without testing in a non-production VPC
- NEVER assume all domains support DNSSEC — most do not
- NEVER disable DNSSEC validation permanently without security team approval
- NEVER ignore SERVFAIL responses — they may indicate DNSSEC issues or actual DNS attacks

## Phase 3 — Rollback

1. Disable DNSSEC validation: `aws route53resolver update-resolver-dnssec-config --resource-id <vpc-id> --validation DISABLE`
2. Verify DNS resolution is restored for previously failing domains
3. Document which domains had DNSSEC issues
4. Plan DNSSEC re-enablement after domain issues are resolved
5. Notify the security team about the DNSSEC status change

## Output Format

```yaml
root_cause: "dnssec_validation — <specific_cause>"
evidence:
  - type: dnssec_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Resolve DNSSEC validation failure"
  long_term: "Monitor DNSSEC health and domain compliance"
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
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"
