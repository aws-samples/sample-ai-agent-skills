---
title: "B2 — Libfabric Issues"
description: "Diagnose libfabric provider and library configuration issues"
status: active
severity: HIGH
triggers: ["libfabric error", "fi_info no provider", "fabric provider not found", "FI_PROVIDER error"]
owner: devops-agent
objective: "Resolve libfabric configuration issues and ensure the EFA provider is functional"
context: "Libfabric is the user-space library that provides the fabric interface for EFA. The EFA provider in libfabric enables OS-bypass networking."
---

## Phase 1 — Triage

MUST:
- List available providers: `fi_info -p efa`
- Check libfabric version: `fi_info --version`
- Verify EFA provider capabilities: `fi_info -p efa -t FI_EP_RDM`

SHOULD:
- Check environment variables: `echo $FI_PROVIDER $FI_LOG_LEVEL`
- Verify library paths: `ldconfig -p | grep libfabric`

MAY:
- Enable debug logging: `FI_LOG_LEVEL=debug fi_info -p efa`
- Check for library version conflicts: `ldd $(which fi_info)`

## Phase 2 — Remediate

MUST:
- Ensure libfabric is installed via the EFA installer (not system package manager)
- Set the EFA provider: `export FI_PROVIDER=efa`
- Verify the EFA kernel module is loaded: `lsmod | grep efa`

SHOULD:
- Set library path if needed: `export LD_LIBRARY_PATH=/opt/amazon/efa/lib64:$LD_LIBRARY_PATH`
- Update to the latest EFA installer for libfabric updates

MAY:
- Compile libfabric from source for custom configurations

## Safety Ratings

```
safety_ratings:
  - "fi_info: GREEN — read-only provider enumeration"
  - "ldconfig -p: GREEN — read-only library cache query"
  - "export FI_PROVIDER: GREEN — sets environment variable for current session"
  - "EFA installer update: YELLOW — updates system libraries"
```

## Escalation Conditions

- Libfabric EFA provider not available despite driver being loaded
- Version conflicts between system libfabric and EFA-installed libfabric
- Custom application requires specific libfabric features not in the EFA version
- Libfabric segfaults during provider initialization

## Data Sensitivity

- `fi_info` output: LOW — lists fabric providers and capabilities
- Library paths: LOW — system library locations
- Environment variables: LOW — runtime configuration
- Debug logs: MEDIUM — may contain network addresses and connection details

## Prohibited Actions

- NEVER install libfabric from the system package manager — it conflicts with the EFA version
- NEVER set FI_PROVIDER to a non-EFA provider as a permanent fix for EFA issues
- NEVER modify files in /opt/amazon/efa/ directly — use the installer for updates
- NEVER ignore libfabric version mismatches between nodes in a cluster

## Phase 3 — Rollback

1. Restore environment variables to defaults: `unset FI_PROVIDER FI_LOG_LEVEL`
2. If libfabric was updated, reinstall the previous EFA installer version
3. Verify library paths are correct: `ldconfig -p | grep libfabric`
4. Test basic connectivity: `fi_pingpong -p efa`
5. Confirm MPI/NCCL workloads function with the restored configuration

## Common Issues

- symptoms: "fi_info -p efa returns 'No data available' or empty output"
  diagnosis: "EFA kernel module is not loaded or EFA interface is not attached."
  resolution: "Load the module: sudo modprobe efa. Verify EFA interface: aws ec2 describe-network-interfaces."

- symptoms: "Application uses wrong libfabric version (system vs EFA)"
  diagnosis: "LD_LIBRARY_PATH does not prioritize the EFA libfabric installation."
  resolution: "Set LD_LIBRARY_PATH=/opt/amazon/efa/lib64:$LD_LIBRARY_PATH before running the application."

## Output Format

```yaml
root_cause: "libfabric — <specific_cause>"
evidence:
  - type: library_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix libfabric provider or library path"
  long_term: "Standardize EFA installation in AMI"
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
