---
title: "A1 — Transfer Family Server Creation Failures"
description: "Diagnose why a Transfer Family server fails to create or start"
status: active
severity: HIGH
triggers:
  - "server creation failed"
  - "server won't start"
  - "create-server error"
  - "server state OFFLINE"
  - "START_FAILED"
owner: devops-agent
objective: "Identify why a Transfer Family server cannot be created or transitions to an error state"
context: "Transfer Family server creation can fail due to IAM role issues, VPC configuration problems, subnet/security group misconfigurations, or service quota limits. The server may create but remain in OFFLINE or START_FAILED state."
---

## Phase 1 — Triage

MUST:
- Check server state: `aws transfer describe-server --server-id <server-id> --query 'Server.State'`
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=transfer.amazonaws.com --max-results 20`
- Verify the IAM logging role trust policy: `aws iam get-role --role-name <logging-role> --query 'Role.AssumeRolePolicyDocument'`
- Check service quotas: `aws service-quotas get-service-quota --service-code transfer --quota-code L-8B27BBE0`
- If VPC endpoint type, verify subnets exist and are in different AZs: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`

SHOULD:
- Check if the VPC endpoint was created: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.transfer.server`
- Verify security groups allow required ports: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Check for conflicting Elastic IP associations if using VPC endpoint with EIPs

MAY:
- Check AWS Health Dashboard for Transfer Family service events
- Review CloudFormation or Terraform stack events if server was created via IaC

## Phase 2 — Remediate

MUST:
- Fix IAM logging role trust policy to include transfer.amazonaws.com as a trusted principal
- Ensure subnets are in supported AZs and have available IP addresses
- Ensure security groups allow inbound traffic on required protocol ports
- Request quota increase if server limit is reached

SHOULD:
- Use VPC endpoint type instead of VPC_ENDPOINT (legacy) for new servers
- Tag servers for identification and cost tracking

MAY:
- Implement CloudFormation/Terraform with proper dependency ordering for VPC resources

## Common Issues

- symptoms: "Server stuck in START_FAILED state"
  diagnosis: "IAM logging role does not trust transfer.amazonaws.com or lacks CloudWatch Logs permissions."
  resolution: "Update the logging role trust policy and attach CloudWatch Logs permissions."

- symptoms: "CreateServer API returns AccessDeniedException"
  diagnosis: "The calling IAM principal lacks transfer:CreateServer permission or there is an SCP deny."
  resolution: "Add transfer:CreateServer to the IAM policy. Check SCPs in AWS Organizations."

- symptoms: "Server creation fails with subnet error"
  diagnosis: "Specified subnets are in the same AZ or do not exist in the VPC."
  resolution: "Provide subnets in different AZs within the same VPC."

## Output Format

```yaml
root_cause: "server_creation_failure — <specific_cause>"
evidence:
  - type: server_state
    content: "<server state and error details>"
  - type: cloudtrail
    content: "<relevant creation events>"
  - type: iam_role
    content: "<trust policy and permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration issue and retry server creation"
  long_term: "Implement IaC templates with validated configurations"
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
