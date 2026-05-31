---
title: "Z1 — General Transfer Family Troubleshooting (Catch-All)"
description: "Fallback SOP for Transfer Family issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Transfer Family issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all servers and their states: `aws transfer list-servers --query 'Servers[*].{ServerId:ServerId,State:State,EndpointType:EndpointType,Protocols:Protocols,IdentityProviderType:IdentityProviderType}'`
- Get full server configuration: `aws transfer describe-server --server-id <server-id>`
- Check CloudWatch transfer metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Transfer --metric-name FilesIn --dimensions Name=ServerId,Value=<server-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check structured logs for errors: `aws logs filter-log-events --log-group-name /aws/transfer/<server-id> --filter-pattern "\"ERROR\"" --start-time <epoch-ms>`
- Test basic connectivity: `nc -zv <endpoint> 22` (SFTP) or `nc -zv <endpoint> 21` (FTP/FTPS)
- List users: `aws transfer list-users --server-id <server-id>`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=transfer.amazonaws.com --max-results 10`
- Check endpoint details: `aws transfer describe-server --server-id <server-id> --query 'Server.EndpointDetails'`
- Check identity provider configuration: `aws transfer describe-server --server-id <server-id> --query 'Server.IdentityProviderDetails'`
- Check workflow associations: `aws transfer describe-server --server-id <server-id> --query 'Server.WorkflowDetails'`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Server creation / startup / endpoint → Server (A1-A3)
- Login / authentication / SSH keys → Authentication (B1-B3)
- Connection timeout / refused / network → Connectivity (C1-C3)
- S3 access denied / EFS permissions → Storage (D1-D2)
- Directory mapping / home directory → Directory (E1-E2)
- Slow transfers / throughput → Performance (F1-F2)
- Host key / encryption / ciphers → Security (G1-G2)
- Workflow execution / post-upload → Workflows (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all server configurations systematically
- Review CloudTrail for recent configuration changes
- Check structured logs for patterns
- Verify IAM roles and trust policies for all users
- Test identity provider if custom IdP: `aws transfer test-identity-provider --server-id <server-id> --user-name <username>`

SHOULD:
- Check AWS Health Dashboard for Transfer Family service events
- Compare with a known-good server configuration
- Check security groups and NACLs for VPC endpoint servers
- Verify DNS resolution and network path

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<server|authentication|connectivity|storage|directory|performance|security|workflows|unknown>"
investigation_path: "server config → CloudWatch → structured logs → <domain_classification>"
evidence:
  - type: server_config
    content: "<server configuration summary>"
  - type: cloudwatch
    content: "<transfer metrics>"
  - type: structured_logs
    content: "<relevant log entries>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
