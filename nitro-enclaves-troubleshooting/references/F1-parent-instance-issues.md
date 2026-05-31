---
title: "F1 — Parent Instance Issues"
description: "Diagnose parent EC2 instance configuration, resource, and compatibility issues affecting enclaves"
status: active
severity: HIGH
triggers: ["instance not supporting enclaves", "parent instance error", "instance type incompatible", "enclave options not enabled"]
owner: devops-agent
objective: "Resolve parent instance configuration issues to enable enclave operations"
context: "The parent instance must be a Nitro-based type with EnclaveOptions enabled at launch. It shares CPU and memory with the enclave. Instance configuration, drivers, and services affect enclave operation."
---

## Phase 1 — Triage

MUST:
- Check enclave support on instance: `aws ec2 describe-instances --instance-ids <id> --query 'Reservations[*].Instances[*].{Type:InstanceType,EnclaveOptions:EnclaveOptions,State:State}'`
- Verify instance type supports enclaves: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[*].{Type:InstanceType,EnclaveSupport:NitroEnclavesSupport}'`
- Check instance health: `aws ec2 describe-instance-status --instance-ids <id>`

SHOULD:
- Verify nitro-cli is installed: `nitro-cli --version`
- Check Nitro Enclaves kernel module: `lsmod | grep nitro_enclaves`
- Verify allocator service: `systemctl status nitro-enclaves-allocator.service`

MAY:
- Check instance system logs: `aws ec2 get-console-output --instance-id <id>`
- Review instance metadata for configuration: `curl -s http://169.254.169.254/latest/meta-data/instance-type`
- Check available instance types with enclave support: `aws ec2 describe-instance-types --filters Name=enclave-support,Values=true --query 'InstanceTypes[*].InstanceType'`

## Phase 2 — Remediate

MUST:
- If enclave support not enabled: launch new instance with `--enclave-options Enabled=true`
- If instance type does not support enclaves: migrate to a supported Nitro-based type
- Install nitro-cli if missing: follow AWS documentation for the instance OS

SHOULD:
- Install and enable allocator service: `sudo systemctl enable --now nitro-enclaves-allocator.service`
- Add user to `ne` group for enclave access: `sudo usermod -aG ne <username>`
- Configure allocator with appropriate memory and CPU settings

MAY:
- Use an AMI with nitro-cli pre-installed (Amazon Linux 2 or AL2023)
- Set up instance user data to automatically configure enclave support on launch
- Create a launch template with enclave options for consistent deployments

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-instances: GREEN — read-only instance info"
  - "ec2 describe-instance-types: GREEN — read-only type info"
  - "ec2 describe-instance-status: GREEN — read-only health check"
  - "ec2 run-instances with enclave options: YELLOW — launches new instance"
  - "ec2 terminate-instances: RED — destroys instance and all local data"
```

## Escalation Conditions

- Instance migration required from non-Nitro to Nitro instance type
- Production instance needs relaunch to enable enclave support
- Instance type with enclave support not available in the required region
- AMI does not include nitro-cli and custom AMI build is needed

## Data Sensitivity

- `aws ec2 describe-instances`: MEDIUM — returns instance metadata including network config
- `aws ec2 get-console-output`: MEDIUM — may contain boot logs with configuration details
- Instance metadata: LOW — returns instance type and configuration
- `/var/log/`: MEDIUM — contains system and application logs

## Prohibited Actions

- NEVER attempt to enable enclave support on a running instance — it requires relaunch
- NEVER terminate a production instance without migration plan
- NEVER use instance types without verifying enclave support
- NEVER skip allocator service configuration — enclaves need hugepages

## Phase 3 — Rollback

1. If new instance was launched incorrectly, terminate and relaunch with correct configuration
2. If allocator was misconfigured, restore settings and restart service
3. If user group membership was changed, verify access permissions
4. Migrate workloads back to original instance if new instance has issues
5. Document the correct instance configuration for future deployments

## Common Issues

- symptoms: "nitro-cli run-enclave fails with 'Could not open /dev/nitro_enclaves'"
  diagnosis: "Nitro Enclaves kernel module is not loaded, or user is not in the 'ne' group."
  resolution: "Load module: sudo modprobe nitro_enclaves. Add user to ne group: sudo usermod -aG ne <user>. Log out and back in."

- symptoms: "Instance type does not show EnclaveSupport: supported"
  diagnosis: "Instance type is not Nitro-based or does not support enclaves (e.g., t3.micro, older generation types)."
  resolution: "Migrate to a supported instance type. Check supported types: aws ec2 describe-instance-types --filters Name=enclave-support,Values=true."

## Output Format

```yaml
root_cause: "parent_instance — <specific_cause>"
evidence:
  - type: instance_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix instance configuration"
  long_term: "Use launch templates with enclave support and pre-configured AMIs"
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
