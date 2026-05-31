---
title: "G1 — Networking and Connectivity"
description: "Diagnose DRS networking and staging area connectivity issues"
status: active
severity: HIGH
triggers:
  - "staging area network"
  - "replication connectivity"
  - "DRS network error"
  - "TCP 1500 blocked"
owner: devops-agent
objective: "Resolve networking issues affecting DRS replication and recovery"
context: "DRS requires specific network paths: source→replication server (TCP 1500), source→DRS API (TCP 443), staging area→S3. Network issues are the most common cause of replication failures."
---

## Phase 1 — Triage

MUST:
- Check staging area subnet config: `aws drs get-replication-configuration --source-server-id <id> --query '{Subnet:stagingAreaSubnetId,SGs:stagingAreaTags}'`
- Verify subnet route table has internet/NAT route: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`
- Check security groups allow TCP 1500 inbound: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify NACLs don't block required traffic

SHOULD:
- Test connectivity from source to replication server IP on TCP 1500
- Verify DNS resolution of DRS endpoints from source
- Check VPC endpoint configuration if using private connectivity

MAY:
- Use VPC Flow Logs to identify blocked traffic
- Check for MTU issues on the network path

## Phase 2 — Remediate

MUST:
- Add route to internet/NAT if missing from staging area subnet
- Open TCP 1500 inbound in staging area security groups
- Ensure source can reach DRS endpoints on TCP 443

SHOULD:
- Create VPC endpoints for S3 and DRS if using private networking
- Document required network paths for DR runbook

MAY:
- Set up VPC Flow Logs for ongoing monitoring
- Configure dedicated replication subnet with optimized routing

## Common Issues

- symptoms: "Replication stalls after VPC security group change"
  diagnosis: "Security group update removed TCP 1500 inbound rule."
  resolution: "Re-add TCP 1500 inbound rule to staging area security group."

## Output Format

```yaml
root_cause: "networking — <specific_cause>"
evidence:
  - type: network_config
    content: "<subnet, route table, security group details>"
severity: HIGH
mitigation:
  immediate: "Fix the specific network path issue"
  long_term: "Document and protect DRS network requirements with SCPs or Config rules"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Network configuration reveals DR infrastructure topology |
| Blast Radius | HIGH — Network issues affect all replication for all source servers |
| Rollback Complexity | LOW — Network configuration changes can be reverted |
| Compliance Scope | HIGH — Network connectivity is required for continuous replication |

## Escalation Conditions

- Escalate IMMEDIATELY to network team if staging area connectivity is lost
- Escalate to DR coordinator if network issues affect replication for multiple servers
- Escalate to security if network changes expose staging area to unauthorized access

## Data Sensitivity

- **Staging area subnet configuration** reveals DR infrastructure
- **Security group rules** define network access patterns
- **Route table entries** reveal network topology

## PROHIBITED Actions

- Do NOT modify staging area security groups without understanding the impact on all source servers
- Do NOT open security groups to 0.0.0.0/0 to fix connectivity
- Do NOT change route tables without verifying the impact on replication traffic
- Do NOT remove VPC endpoints without confirming alternative connectivity exists

## Phase 3 — Rollback

- Restore previous security group rules
- Restore previous route table entries
- Restore previous NACL rules
- Verify replication resumes for all source servers after network rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
