---
title: "J3 — Graviton / ARM Architecture Issues"
description: "Diagnose issues specific to Graviton (ARM) instances"
status: active
severity: MEDIUM
triggers:
  - "arm64.*incompatible"
  - "Graviton.*error"
  - "architecture.*mismatch"
  - "exec format error"
owner: devops-agent
objective: "Resolve ARM architecture compatibility issues"
context: "Graviton instances (*.g suffix: m6g, c7g, t4g, etc.) use ARM64 architecture. Software must be compiled for arm64. x86_64 binaries will not run natively. Docker images must be multi-arch or arm64-specific."
---

## Phase 1 — Triage

MUST:
- Confirm instance architecture: `aws ec2 describe-instance-types --instance-types <type>` → ProcessorInfo.SupportedArchitectures
- Check AMI architecture matches: `aws ec2 describe-images --image-ids <ami-id>` → Architecture
- If application fails: check for x86_64 binaries on arm64 instance (`file <binary>`)

SHOULD:
- Check Docker images for multi-arch support
- Verify all dependencies have arm64 builds

## Common Issues

- symptoms: "'exec format error' when running application"
  diagnosis: "x86_64 binary on arm64 instance. Binary must be recompiled for arm64."
  resolution: "Use arm64 build of the application. For Docker, use multi-arch images."

- symptoms: "Docker container fails to start on Graviton"
  diagnosis: "Docker image is x86_64 only, no arm64 variant."
  resolution: "Build multi-arch Docker image or use arm64-specific image tag."

- symptoms: "Package installation fails on Graviton"
  diagnosis: "Package repository doesn't have arm64 builds for the required package."
  resolution: "Check for arm64/aarch64 package availability. May need to compile from source."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instance-types, describe-images Architecture: GREEN — read-only"
  - "file <binary> to check architecture via SSM: GREEN — read-only"
  - "Build multi-arch Docker image: GREEN — creates new image, no impact"
  - "Install arm64 packages: YELLOW — modifies OS packages, recoverable"
  - "Recompile application for arm64: YELLOW — requires build pipeline changes"
  - "Change instance type back to x86: YELLOW — requires stop+start, recoverable"
```

## Escalation Conditions
- Critical application binary has no arm64 build available
- Third-party software vendor does not provide arm64 packages
- Docker base images do not have arm64 variants
- Migration to Graviton requires recompilation of proprietary software
- Performance regression on Graviton compared to x86 for specific workloads

## Data Sensitivity
- LOW: describe-instance-types (public architecture information)
- LOW: describe-images Architecture (public AMI metadata)
- MEDIUM: file command output via SSM (reveals binary details and build information)

## Prohibited Actions
- NEVER suggest running x86_64 binaries on Graviton without emulation (will fail with exec format error)
- NEVER suggest using QEMU emulation for production workloads (significant performance penalty)
- NEVER suggest migrating to Graviton without testing all application dependencies for arm64 compatibility
- NEVER suggest arm64 as a drop-in replacement without validating performance characteristics

## Phase 3 — Rollback
- If instance type was changed to Graviton: stop, change back to x86 instance type, start
- If application was recompiled for arm64: deploy previous x86_64 build
- If Docker images were changed to multi-arch: revert to x86_64-specific image tags
- If packages were installed for arm64: revert to x86 instance and reinstall x86_64 packages

## Output Format

```yaml
root_cause: "<binary_architecture|docker_image|package_unavailable|ami_mismatch>"
evidence:
  - type: architecture_check
    content: "<file command output or Docker image manifest>"
severity: MEDIUM
mitigation:
  immediate: "Use arm64 compatible binaries/images"
  long_term: "Build CI/CD pipeline for multi-arch, test on Graviton in staging"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
