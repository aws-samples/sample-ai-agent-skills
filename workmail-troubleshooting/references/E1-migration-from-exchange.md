---
title: "E1 — Migration from Exchange"
description: "Diagnose WorkMail migration from Microsoft Exchange issues"
status: active
severity: HIGH
triggers: ["migration failed", "Exchange migration error", "migration tool issue", "mailbox migration stuck"]
owner: devops-agent
objective: "Resolve Exchange to WorkMail migration issues including tool configuration, connectivity, and data transfer"
context: "Migration from Exchange uses the WorkMail Migration Tool which connects via EWS. Migration requires careful DNS cutover planning."
---

## Phase 1 — Triage

MUST:
- Verify WorkMail organization is active: `aws workmail describe-organization --organization-id <org-id>`
- Verify target users are created and registered in WorkMail: `aws workmail list-users --organization-id <org-id>`
- Check migration tool connectivity to source Exchange server
- Verify EWS endpoint is accessible from the migration environment

SHOULD:
- Check migration tool logs for specific error messages
- Verify source Exchange mailbox permissions (impersonation rights)
- Confirm network connectivity between migration tool and both Exchange and WorkMail

MAY:
- Test migration with a single pilot mailbox before bulk migration
- Check Exchange server health and EWS availability

## Phase 2 — Remediate

MUST:
- Fix EWS connectivity issues (firewall, proxy, SSL certificates)
- Grant impersonation rights on the source Exchange server
- Ensure target WorkMail users match source Exchange mailboxes

SHOULD:
- Migrate in batches to manage bandwidth and reduce risk
- Plan DNS MX record cutover for minimal email disruption
- Test migrated mailboxes before cutting over DNS

MAY:
- Set up coexistence routing during migration period
- Configure email forwarding from Exchange to WorkMail during transition

## Safety Ratings

```
safety_ratings:
  - "workmail describe-organization: GREEN — read-only organization inspection"
  - "workmail list-users: GREEN — read-only user listing"
  - "Migration tool execution: YELLOW — copies email data between systems"
  - "DNS MX record cutover: RED — changes email routing for the entire domain"
```

## Escalation Conditions

- Migration tool cannot connect to Exchange despite correct configuration
- Large mailboxes fail to migrate due to timeout or size limits
- Data integrity issues found in migrated mailboxes
- DNS cutover causes email delivery failures

## Data Sensitivity

- `aws workmail describe-organization`: LOW — returns organization metadata
- `aws workmail list-users`: MEDIUM — reveals user identities
- Migration data: HIGH — contains all email content being migrated
- Exchange impersonation credentials: HIGH — provides access to all Exchange mailboxes

## Prohibited Actions

- NEVER cut over DNS MX records before verifying migration completeness
- NEVER delete source Exchange mailboxes before confirming successful migration
- NEVER run migration during peak business hours without approval
- NEVER store Exchange impersonation credentials in plain text

## Phase 3 — Rollback

1. If DNS was cut over prematurely, revert MX records to point to Exchange
2. If migrated data is incorrect, delete migrated content and re-run migration
3. If WorkMail users were created incorrectly, deregister and recreate them
4. Restore Exchange as primary email system if migration cannot be completed
5. Document migration issues and update the migration plan before retry

## Common Issues

- symptoms: "Migration tool fails with 'Cannot connect to EWS'"
  diagnosis: "EWS endpoint is not accessible from the migration environment."
  resolution: "Verify EWS URL, check firewall rules, ensure SSL certificate is valid. Test: curl https://<exchange>/EWS/Exchange.asmx."

- symptoms: "Migration completes but some emails are missing"
  diagnosis: "Large attachments or specific message types may fail to migrate."
  resolution: "Check migration tool logs for skipped items. Re-run migration for failed items. Verify message size limits."

## Output Format

```yaml
root_cause: "exchange_migration — <specific_cause>"
evidence:
  - type: migration_error
    content: "<migration tool status, connectivity, data transfer details>"
severity: HIGH
mitigation:
  immediate: "Fix migration issue"
  long_term: "Document migration runbook with tested procedures"
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
