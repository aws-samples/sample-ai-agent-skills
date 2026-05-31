---
title: "B1 — Enclave Start Errors"
description: "Diagnose enclave startup failures after successful creation"
status: active
severity: HIGH
triggers: ["enclave start failed", "enclave crashed on start", "enclave not running", "enclave terminated unexpectedly"]
owner: devops-agent
objective: "Resolve enclave startup errors and achieve stable enclave operation"
context: "Enclave startup failures occur after run-enclave is accepted but the enclave fails to initialize. Causes include insufficient resources, application errors, or EIF issues. Console output is the primary diagnostic tool."
---

## Phase 1 — Triage

MUST:
- Check enclave status: `nitro-cli describe-enclaves`
- View enclave console output: `nitro-cli console --enclave-id <id>`
- Check resource allocation: verify memory and CPU count used in run-enclave command

SHOULD:
- Check parent instance available resources: `free -m` and `nproc`
- Review nitro enclaves log: `cat /var/log/nitro_enclaves/nitro_enclaves.log`
- Verify EIF integrity: `nitro-cli describe-eif --eif-path <path>`

MAY:
- Check kernel messages: `dmesg | tail -50`
- Review allocator configuration: `cat /etc/nitro_enclaves/allocator.yaml`
- Check if OOM killer terminated the enclave

## Phase 2 — Remediate

MUST:
- Ensure allocated memory is sufficient for the application (EIF size + runtime memory)
- Ensure allocated CPU count is at least 2 (minimum for enclave)
- Fix application errors visible in console output

SHOULD:
- Increase memory allocation if application runs out of memory
- Verify application handles the no-network environment gracefully
- Ensure application writes diagnostic output to stdout for console visibility

MAY:
- Test with minimal EIF to isolate application vs infrastructure issues
- Adjust allocator hugepages configuration for larger memory requirements
- Profile application memory usage in Docker before building EIF

## Safety Ratings

```
safety_ratings:
  - "describe-enclaves: GREEN — read-only status check"
  - "console: GREEN — read-only console output"
  - "describe-eif: GREEN — read-only EIF inspection"
  - "run-enclave: YELLOW — allocates resources from parent"
  - "terminate-enclave: YELLOW — destroys enclave state"
```

## Escalation Conditions

- Enclave crashes repeatedly with no useful console output
- Application requires more memory than the instance can provide
- Enclave starts but application fails due to missing runtime dependencies
- Kernel-level enclave errors requiring AMI or driver updates

## Data Sensitivity

- `nitro-cli describe-enclaves`: LOW — returns enclave metadata
- `nitro-cli console`: MEDIUM — may show application output including processed data references
- `/var/log/nitro_enclaves/`: MEDIUM — contains enclave lifecycle events
- `dmesg`: LOW — contains kernel messages

## Prohibited Actions

- NEVER allocate all parent instance memory to the enclave — leave at least 512MB for parent
- NEVER ignore console output errors — they are the primary diagnostic source
- NEVER repeatedly restart a crashing enclave without investigating the root cause
- NEVER assume enclave startup is instant — allow time for application initialization

## Phase 3 — Rollback

1. Terminate the failing enclave: `nitro-cli terminate-enclave --enclave-id <id>`
2. If resource allocation was wrong, restart with corrected values
3. If EIF is corrupted, rebuild from Docker image
4. Verify parent instance health after enclave termination
5. Test with known-good EIF to confirm infrastructure is working

## Common Issues

- symptoms: "Enclave starts but immediately shows TERMINATED status"
  diagnosis: "Application crashes on startup, typically due to missing dependencies or insufficient memory."
  resolution: "Check console output for crash details. Increase memory allocation. Verify Docker image runs correctly outside enclave."

- symptoms: "Enclave console shows 'out of memory' errors"
  diagnosis: "Allocated memory is insufficient for the application's runtime requirements."
  resolution: "Increase --memory parameter in run-enclave. Profile application memory usage. Optimize application memory footprint."

## Output Format

```yaml
root_cause: "enclave_start — <specific_cause>"
evidence:
  - type: start_error
    content: "<error details from console>"
severity: HIGH
mitigation:
  immediate: "Fix startup issue and restart enclave"
  long_term: "Implement enclave health monitoring and auto-recovery"
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
