---
title: "D1 — Key Alias Issues"
description: "Diagnose key alias creation, resolution, and management issues"
status: active
severity: MEDIUM
triggers: ["alias not found", "alias error", "create-alias failed", "alias resolution"]
owner: devops-agent
objective: "Resolve key alias issues in Payment Cryptography"
context: "Aliases provide friendly names for keys. Aliases can be reassigned. Some APIs accept aliases, others require ARNs."
---

## Phase 1 — Triage

MUST:
- List aliases: `aws payment-cryptography list-aliases`
- Get alias details: `aws payment-cryptography get-alias --alias-name alias/<name>`
- Verify the alias points to the expected key

SHOULD:
- Check if alias was recently reassigned to a different key
- Verify alias name format (must start with 'alias/')

MAY:
- Review CloudTrail for alias modification events
- Check if the target key is enabled

## Phase 2 — Remediate

MUST:
- Create missing aliases: `aws payment-cryptography create-alias --alias-name alias/<name> --key-arn <key-arn>`
- Fix alias pointing to wrong key: `aws payment-cryptography update-alias --alias-name alias/<name> --key-arn <correct-key-arn>`
- Use full key ARN if alias resolution is unreliable

SHOULD:
- Implement alias naming conventions
- Document alias-to-key mappings

MAY:
- Set up alias change monitoring
- Implement alias validation in deployment pipelines

## Safety Ratings

```
safety_ratings:
  - "list-aliases: GREEN — read-only alias enumeration"
  - "get-alias: GREEN — read-only alias details"
  - "create-alias: YELLOW — creates new alias mapping"
  - "update-alias: RED — changes which key an alias points to"
```

## Escalation Conditions

- Alias reassignment affected production operations
- Multiple aliases point to the same key causing confusion
- Alias naming conflicts between teams
- Alias changes not tracked in change management

## Data Sensitivity

- `aws payment-cryptography list-aliases`: LOW — returns alias names and key ARNs
- `aws payment-cryptography get-alias`: LOW — returns alias-to-key mapping
- Alias names: LOW — friendly names for keys

## Prohibited Actions

- NEVER reassign aliases in production without change management approval
- NEVER delete aliases used by production applications
- NEVER create aliases that obscure key purpose
- NEVER use aliases as the sole key identifier in critical systems

## Phase 3 — Rollback

1. If alias was reassigned incorrectly: update to point to correct key: `aws payment-cryptography update-alias --alias-name alias/<name> --key-arn <correct-arn>`
2. If alias was deleted: recreate: `aws payment-cryptography create-alias --alias-name alias/<name> --key-arn <key-arn>`
3. If wrong alias was created: delete it: `aws payment-cryptography delete-alias --alias-name alias/<name>`
4. Verify all applications using the alias are functioning correctly
5. Document alias changes for operations team

## Common Issues

- symptoms: "API returns ResourceNotFoundException for alias"
  diagnosis: "Alias does not exist or alias name format is incorrect."
  resolution: "Verify alias name starts with 'alias/'. Create if missing: aws payment-cryptography create-alias."

- symptoms: "Operations return wrong results after alias change"
  diagnosis: "Alias was reassigned to a different key."
  resolution: "Check alias-to-key mapping. Update alias to point to correct key or update application to use key ARN directly."

## Output Format

```yaml
root_cause: "key_alias — <specific_cause>"
evidence:
  - type: alias_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix alias mapping"
  long_term: "Implement alias management standards"
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
