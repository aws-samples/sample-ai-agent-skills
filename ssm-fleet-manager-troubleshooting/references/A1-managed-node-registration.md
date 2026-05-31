---
title: "A1 — Managed Node Registration"
description: "Diagnose managed node registration failures for EC2 instances"
status: active
severity: HIGH
triggers: ["instance not registering", "node not managed", "SSM registration failed"]
owner: devops-agent
objective: "Resolve EC2 instance registration with Systems Manager"
context: "EC2 instances register via IAM instance profile and SSM Agent. Registration requires network connectivity to SSM endpoints."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name,Profile:IamInstanceProfile.Arn}'`
- Verify SSM Agent registration: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- Check IAM instance profile: `aws iam get-instance-profile --instance-profile-name <profile-name>`

SHOULD:
- Verify the instance profile role has `AmazonSSMManagedInstanceCore` policy attached
- Check SSM Agent logs on the instance via EC2 console or serial console

MAY:
- Review CloudTrail for RegisterManagedInstance events
- Check if the instance is in a private subnet without SSM VPC endpoints

## Phase 2 — Remediate

MUST:
- Attach IAM instance profile if missing: `aws ec2 associate-iam-instance-profile --instance-id <instance-id> --iam-instance-profile Name=<profile-name>`
- Ensure `AmazonSSMManagedInstanceCore` policy is attached to the role
- Verify SSM Agent is installed and running on the instance

SHOULD:
- Create VPC endpoints if instance is in a private subnet without NAT
- Update SSM Agent to latest version: `aws ssm send-command --instance-ids <instance-id> --document-name AWS-UpdateSSMAgent`

MAY:
- Configure SSM Agent auto-update for the account

## Common Issues

- symptoms: "Instance running but not appearing in Fleet Manager"
  diagnosis: "Missing IAM instance profile or incorrect permissions."
  resolution: "Attach instance profile with AmazonSSMManagedInstanceCore policy."

- symptoms: "Instance profile attached but SSM Agent shows as Connection Lost"
  diagnosis: "Network connectivity issue — instance cannot reach SSM endpoints."
  resolution: "Add VPC endpoints for ssm, ssmmessages, ec2messages or configure NAT gateway."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only query of managed node status"
  - "associate-iam-instance-profile: YELLOW — modifies instance IAM configuration"
  - "send-command AWS-UpdateSSMAgent: YELLOW — updates agent software on instance"
  - "ec2 describe-instances: GREEN — read-only instance metadata query"
```

## Escalation Conditions

- Instance profile and network are correct but node still not registering after 15 minutes
- SSM Agent crashes repeatedly on startup (check agent logs)
- Registration fails with internal service errors from SSM
- Custom AMI with modified SSM Agent configuration

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — returns instance metadata and agent version
- `aws ec2 describe-instances`: MEDIUM — may reveal VPC, subnet, and security group details
- `aws iam get-instance-profile`: MEDIUM — reveals IAM role and policy configuration
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions

## Prohibited Actions

- NEVER remove an existing IAM instance profile without confirming it is not used by other services
- NEVER install SSM Agent from untrusted sources — use only official AWS packages
- NEVER disable SSM Agent on production instances to test registration
- NEVER modify VPC route tables without understanding the full network topology

## Phase 3 — Rollback

1. If an incorrect instance profile was attached, disassociate it: `aws ec2 disassociate-iam-instance-profile --association-id <association-id>`
2. If SSM Agent was updated and caused issues, downgrade: `aws ssm send-command --instance-ids <instance-id> --document-name AWS-UpdateSSMAgent --parameters version=<previous-version>`
3. If VPC endpoints were created incorrectly, delete them: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <endpoint-id>`
4. Verify the instance returns to its previous operational state
5. Document the rollback action and notify the operations team

## Output Format

```yaml
root_cause: "managed_node_registration — <specific_cause>"
evidence:
  - type: registration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix registration prerequisite"
  long_term: "Enforce instance profile via launch template"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
