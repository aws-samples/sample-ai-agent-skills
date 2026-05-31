---
title: "A1 — External Instance Registration"
description: "Diagnose external instance registration failures with ECS Anywhere"
status: active
severity: HIGH
triggers: ["cannot register instance", "registration failed", "instance not appearing in cluster", "activation code error"]
owner: devops-agent
objective: "Resolve external instance registration issues with ECS Anywhere"
context: "External instances register via SSM activation, then ECS Agent registers with the ECS cluster. Both steps must succeed."
---

## Phase 1 — Triage

MUST:
- Check SSM activations: `aws ssm describe-activations --query 'ActivationList[*].{Id:ActivationId,Expired:Expired,RegistrationLimit:RegistrationLimit,RegistrationsCount:RegistrationsCount}'`
- Check if instance appears in SSM: `aws ssm describe-instance-information --filters Key=ResourceType,Values=ManagedInstance`
- Check ECS cluster for the instance: `aws ecs list-container-instances --cluster <cluster-name> --status ACTIVE`

SHOULD:
- Verify the activation code has not expired and has remaining registrations
- Check if SSM Agent is installed and running on the external instance
- Verify Docker/containerd is running on the instance

MAY:
- Check the ECS Agent logs on the instance: `/var/log/ecs/ecs-agent.log`
- Check SSM Agent logs: `/var/log/amazon/ssm/amazon-ssm-agent.log`
- Verify network connectivity from the instance to AWS endpoints

## Phase 2 — Remediate

MUST:
- Create new SSM activation if expired: `aws ssm create-activation --iam-role <ecs-anywhere-role> --registration-limit <count> --default-instance-name <name>`
- Install SSM Agent if missing on the instance
- Install Docker/containerd if missing on the instance
- Run the ECS Anywhere install script with the activation code and ID

SHOULD:
- Verify outbound HTTPS (443) connectivity to SSM, ECS, ECR, and S3 endpoints
- Ensure the IAM role for the activation has the required ECS and SSM permissions
- Tag the instance for identification

MAY:
- Automate registration with configuration management tools (Ansible, Puppet)
- Set up monitoring for instance registration status

## Common Issues

- symptoms: "Instance registers with SSM but does not appear in ECS cluster"
  diagnosis: "ECS Agent is not installed or not running. SSM registration succeeded but ECS registration did not."
  resolution: "Verify ECS Agent is running. Check /var/log/ecs/ecs-agent.log for errors. Re-run the ECS install script."

- symptoms: "Registration fails with 'ActivationExpired' error"
  diagnosis: "The SSM activation code has expired."
  resolution: "Create a new activation: aws ssm create-activation --iam-role <role> --registration-limit <count>."

## Safety Ratings

```
safety_ratings:
  - "ssm describe-activations: GREEN — read-only"
  - "ssm describe-instance-information: GREEN — read-only"
  - "ecs list-container-instances: GREEN — read-only"
  - "ssm create-activation: YELLOW — creates activation credentials"
  - "ssm send-command: RED — executes commands on remote instances"
```

## Escalation Conditions

- Instance registers with SSM but ECS Agent repeatedly fails to connect
- Network firewall rules cannot be modified to allow AWS endpoint access
- IAM role for activation requires permissions that violate organizational policies
- Multiple instances fail registration simultaneously suggesting infrastructure issue

## Data Sensitivity

- `aws ssm describe-activations`: MEDIUM — reveals activation IDs and IAM roles
- `aws ssm describe-instance-information`: MEDIUM — reveals instance IPs and platform details
- `aws ecs list-container-instances`: LOW — instance ARNs only
- SSM activation codes: HIGH — can be used to register new instances
- Instance logs: HIGH — may contain credentials and configuration details

## Prohibited Actions

- NEVER share SSM activation codes in logs, tickets, or chat
- NEVER create activations with unlimited registration limits
- NEVER register instances to production clusters without testing
- NEVER skip Docker/containerd installation before running the ECS install script

## Phase 3 — Rollback

1. If an instance was registered incorrectly, deregister it: `aws ecs deregister-container-instance --cluster <cluster-name> --container-instance <arn> --force`
2. Deregister from SSM: `aws ssm deregister-managed-instance --instance-id <mi-id>`
3. Delete unused activations: `aws ssm delete-activation --activation-id <id>`
4. Verify the instance is removed from both SSM and ECS
5. Document the registration issue and update the registration procedure

## Output Format

```yaml
root_cause: "registration — <specific_cause>"
evidence:
  - type: registration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix registration prerequisites"
  long_term: "Automate registration with validation checks"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
