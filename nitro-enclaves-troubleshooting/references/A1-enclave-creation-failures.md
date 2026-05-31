---
title: "A1 — Enclave Creation Failures"
description: "Diagnose enclave creation and run-enclave command failures"
status: active
severity: HIGH
triggers: ["enclave creation failed", "run-enclave error", "cannot create enclave", "enclave not starting"]
owner: devops-agent
objective: "Resolve enclave creation failures and enable enclave launch"
context: "Enclaves are created using nitro-cli run-enclave on a Nitro-based instance with enclave support enabled. Failures can be caused by missing enclave support, resource constraints, or EIF issues."
---

## Phase 1 — Triage

MUST:
- Check if enclave support is enabled: `aws ec2 describe-instances --instance-ids <id> --query 'Reservations[*].Instances[*].EnclaveOptions'`
- Check for existing running enclave: `nitro-cli describe-enclaves`
- Verify instance type supports enclaves: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[*].NitroEnclavesSupport'`

SHOULD:
- Check nitro-cli is installed: `nitro-cli --version`
- Verify the Nitro Enclaves allocator service is running: `systemctl status nitro-enclaves-allocator.service`
- Check EIF file exists and is readable: `ls -la <path-to-eif>`

MAY:
- Review system logs for enclave errors: `dmesg | grep -i nitro`
- Check nitro enclaves log: `cat /var/log/nitro_enclaves/nitro_enclaves.log`
- Verify hugepages allocation for enclave memory

## Phase 2 — Remediate

MUST:
- Ensure instance has EnclaveOptions.Enabled: true (requires relaunch if not)
- Terminate any existing enclave before starting a new one: `nitro-cli terminate-enclave --enclave-id <id>`
- Verify EIF file is valid: `nitro-cli describe-eif --eif-path <path>`

SHOULD:
- Start and enable the allocator service: `sudo systemctl enable --now nitro-enclaves-allocator.service`
- Configure allocator with appropriate memory: edit `/etc/nitro_enclaves/allocator.yaml`
- Ensure the user is in the `ne` group for enclave access

MAY:
- Rebuild EIF if corrupted: `nitro-cli build-enclave --docker-uri <image> --output-file <output.eif>`
- Check for kernel module: `lsmod | grep nitro_enclaves`
- Update nitro-cli to latest version

## Safety Ratings

```
safety_ratings:
  - "describe-enclaves: GREEN — read-only enumeration"
  - "describe-eif: GREEN — read-only EIF inspection"
  - "ec2 describe-instances: GREEN — read-only instance check"
  - "run-enclave: YELLOW — allocates CPU and memory from parent instance"
  - "terminate-enclave: YELLOW — destroys enclave and all in-memory state"
```

## Escalation Conditions

- Instance type does not support enclaves requiring instance migration
- Enclave support not enabled requiring instance relaunch
- Kernel module missing requiring AMI rebuild or driver installation
- Persistent enclave creation failures after all configuration fixes

## Data Sensitivity

- `aws ec2 describe-instances`: MEDIUM — returns instance metadata
- `nitro-cli describe-enclaves`: LOW — returns enclave metadata
- `nitro-cli describe-eif`: LOW — returns EIF metadata and PCR values
- `/var/log/nitro_enclaves/`: MEDIUM — may contain error details

## Prohibited Actions

- NEVER attempt to enable enclave support on a running instance — it requires relaunch
- NEVER allocate all instance CPU/memory to the enclave — leave resources for the parent
- NEVER run enclaves on non-Nitro instance types — they are not supported
- NEVER ignore EIF validation errors — they indicate image corruption or build issues

## Phase 3 — Rollback

1. If enclave was started with wrong configuration, terminate: `nitro-cli terminate-enclave --enclave-id <id>`
2. If allocator configuration was changed, restore previous settings and restart service
3. If instance needs relaunch for enclave support, plan migration with minimal downtime
4. Verify parent instance is healthy after enclave termination
5. Document the failure cause for future enclave deployments

## Common Issues

- symptoms: "run-enclave returns 'Enclave support is not enabled on this instance'"
  diagnosis: "Instance was launched without EnclaveOptions.Enabled: true."
  resolution: "Launch a new instance with --enclave-options Enabled=true. Migrate workloads from the old instance."

- symptoms: "run-enclave returns 'An enclave is already running'"
  diagnosis: "Only one enclave can run per instance. An existing enclave must be terminated first."
  resolution: "Terminate existing enclave: nitro-cli terminate-enclave --all. Then retry run-enclave."

## Output Format

```yaml
root_cause: "enclave_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix enclave creation prerequisites"
  long_term: "Automate enclave deployment with proper instance configuration"
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
