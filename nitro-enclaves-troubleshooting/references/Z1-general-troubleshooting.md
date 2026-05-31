---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Nitro Enclaves issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["enclave error", "unknown enclave issue", "general enclave problem", "Nitro Enclaves not working"]
owner: devops-agent
objective: "Systematically diagnose and resolve general Nitro Enclaves issues"
context: "This runbook covers general troubleshooting when the specific issue category is unclear. Start here and route to specific runbooks as the problem domain becomes clear."
---

## Phase 1 — Triage

MUST:
- Check enclave status: `nitro-cli describe-enclaves`
- Verify instance supports enclaves: `aws ec2 describe-instances --instance-ids <id> --query 'Reservations[*].Instances[*].EnclaveOptions'`
- Check nitro-cli version: `nitro-cli --version`

SHOULD:
- Review enclave console output: `nitro-cli console --enclave-id <id>` (if enclave is running)
- Check system logs: `dmesg | grep -i nitro`
- Verify allocator service: `systemctl status nitro-enclaves-allocator.service`

MAY:
- Review nitro enclaves log: `cat /var/log/nitro_enclaves/nitro_enclaves.log`
- Check CloudTrail for related API events
- Verify instance health: `aws ec2 describe-instance-status --instance-ids <id>`

## Phase 2 — Remediate

MUST:
- Ensure instance has enclave support enabled
- Verify nitro-cli and allocator service are properly installed and configured
- Check enclave console output for application-level errors

SHOULD:
- Route to specific runbook once issue domain is identified (A1-F1)
- Verify all prerequisites: Nitro instance, enclave enabled, nitro-cli installed, allocator running
- Check if the issue is reproducible with a simple test enclave

MAY:
- Build and run the hello-world sample enclave to validate infrastructure
- Update nitro-cli to latest version
- Check AWS documentation for known issues or recent changes

## Safety Ratings

```
safety_ratings:
  - "describe-enclaves: GREEN — read-only status check"
  - "ec2 describe-instances: GREEN — read-only instance info"
  - "nitro-cli --version: GREEN — read-only version check"
  - "console: GREEN — read-only console output"
  - "dmesg: GREEN — read-only kernel messages"
```

## Escalation Conditions

- Multiple enclave operations failing simultaneously
- Kernel-level errors suggesting driver or hardware issues
- Enclave failures after instance or AMI updates
- Security concerns about enclave isolation integrity

## Data Sensitivity

- `nitro-cli describe-enclaves`: LOW — returns enclave metadata
- `nitro-cli console`: MEDIUM — may show application output
- `dmesg`: LOW — contains kernel messages
- `/var/log/nitro_enclaves/`: MEDIUM — contains enclave lifecycle events

## Prohibited Actions

- NEVER modify kernel modules or drivers without understanding the impact on enclave isolation
- NEVER run untrusted EIF images for troubleshooting
- NEVER disable security features (SELinux, etc.) to work around enclave issues
- NEVER share enclave console output that may contain sensitive application data

## Phase 3 — Rollback

1. If any configuration was changed during troubleshooting, restore previous settings
2. If test enclaves were created, terminate them: `nitro-cli terminate-enclave --all`
3. Restart allocator service if it was modified: `sudo systemctl restart nitro-enclaves-allocator.service`
4. Document findings and route to appropriate specific runbook
5. Verify parent instance is in a clean state after troubleshooting

## Common Issues

- symptoms: "nitro-cli command not found"
  diagnosis: "nitro-cli is not installed on the instance."
  resolution: "Install nitro-cli: sudo amazon-linux-extras install aws-nitro-enclaves-cli (Amazon Linux 2) or sudo dnf install aws-nitro-enclaves-cli (AL2023)."

- symptoms: "Enclave operations fail with various errors on a new instance"
  diagnosis: "Instance was not fully configured for enclaves: missing enclave support, allocator not configured, user not in ne group."
  resolution: "Follow the complete setup: enable enclave options at launch, install nitro-cli, configure allocator, add user to ne group, start allocator service."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Route to specific runbook for comprehensive resolution"
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
