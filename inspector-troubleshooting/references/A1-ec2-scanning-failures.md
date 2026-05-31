---
title: "A1 — EC2 Scanning Failures"
description: "Diagnose why EC2 instances are not being scanned by Inspector"
status: active
severity: HIGH
triggers:
  - "EC2 not scanned"
  - "instance not covered"
  - "EC2 scanning failed"
  - "no findings for instance"
owner: devops-agent
objective: "Identify why EC2 instances are not being scanned and restore scanning coverage"
context: "EC2 scanning requires Inspector v2 enabled with EC2 scan type, SSM agent running on the instance, proper IAM role attached, and the instance appearing as managed in Systems Manager. Missing any of these prevents scanning."
---

## Phase 1 — Triage

MUST:
- Check Inspector account status for EC2: `aws inspector2 batch-get-account-status --query 'Accounts[*].{Account:AccountId,EC2:ResourceState.Ec2.Status}'`
- Check instance coverage: `aws inspector2 list-coverage --filter-criteria '{"resourceId":[{"comparison":"EQUALS","value":"<instance-id>"}]}'`
- Check SSM agent status: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id> --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,AgentVersion:AgentVersion,PlatformType:PlatformType}'`
- Check instance IAM role: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].IamInstanceProfile'`

SHOULD:
- Check if instance is in a running state: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].State'`
- Verify IAM role has AmazonSSMManagedInstanceCore policy
- Check for tag-based exclusion rules in Inspector

MAY:
- Check SSM agent logs on the instance for errors
- Verify VPC endpoints or internet access for SSM connectivity

## Phase 2 — Remediate

MUST:
- Enable EC2 scanning if not active: `aws inspector2 enable --resource-types EC2`
- Install/start SSM agent on the instance
- Attach IAM role with AmazonSSMManagedInstanceCore policy

SHOULD:
- Verify SSM connectivity after fixing: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- Check coverage after fix: `aws inspector2 list-coverage --filter-criteria '{"resourceId":[{"comparison":"EQUALS","value":"<instance-id>"}]}'`

MAY:
- Set up VPC endpoints for SSM if instances are in private subnets
- Configure tag-based inclusion rules for targeted scanning

## Common Issues

- symptoms: "Instance shows NOT_COVERED in Inspector"
  diagnosis: "SSM agent not running or instance not registered as managed."
  resolution: "Install/restart SSM agent. Attach IAM role with SSM permissions."

- symptoms: "EC2 scanning enabled but no instances scanned"
  diagnosis: "No instances have SSM agent running with proper IAM role."
  resolution: "Deploy SSM agent and IAM role to all target instances."

- symptoms: "Instance was scanned before but stopped"
  diagnosis: "SSM agent stopped or IAM role was removed."
  resolution: "Restart SSM agent. Verify IAM role is still attached."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable EC2 scanning | GREEN | Enables vulnerability detection, non-destructive |
| Install/start SSM agent | YELLOW | Agent installation on running instances |
| Attach IAM role with SSM permissions | YELLOW | IAM changes affect instance capabilities |
| Set up VPC endpoints for SSM | GREEN | Additive network configuration |

## Escalation Conditions

- Production instances not being scanned — escalate to security operations
- SSM agent deployment across fleet — escalate to infrastructure team
- IAM role changes on production instances — escalate to operations team
- Scanning configuration changes affecting organization — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status per resource type |
| `aws inspector2 list-coverage` | MEDIUM | Exposes instance coverage and scan status |
| `aws ssm describe-instance-information` | MEDIUM | Exposes instance agent status and platform details |
| `aws ec2 describe-instances` (IAM profile) | MEDIUM | Exposes instance IAM role associations |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** remove SSM agent from instances to resolve scanning issues
- **NEVER** detach IAM roles from production instances without a replacement

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled EC2 scanning | Disable EC2 scanning: `aws inspector2 disable --resource-types EC2` |
| Installed SSM agent | Uninstall SSM agent (not recommended for production) |
| Attached IAM role | Detach IAM role (caution: may affect other SSM functionality) |
| Created VPC endpoints | Delete VPC endpoints: `aws ec2 delete-vpc-endpoints` |

## Output Format

```yaml
root_cause: "ec2_scanning — <specific_cause>"
evidence:
  - type: account_status
    content: "<Inspector EC2 scanning status>"
  - type: coverage
    content: "<instance coverage status>"
  - type: ssm_status
    content: "<SSM agent status>"
severity: HIGH
mitigation:
  immediate: "Fix SSM agent and IAM role on affected instances"
  long_term: "Automate SSM agent deployment and IAM role attachment"
```

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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
