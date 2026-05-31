---
title: "F1 — Security Hardening"
description: "Diagnose Bottlerocket security hardening, SELinux, and compliance issues"
status: active
severity: HIGH
triggers: ["selinux denial", "security compliance", "hardening configuration", "cis benchmark", "security audit"]
owner: devops-agent
objective: "Resolve Bottlerocket security hardening issues and ensure compliance posture"
context: "Bottlerocket is hardened by default with SELinux enforcing, dm-verity, read-only root, and minimal attack surface. Custom workloads may conflict with these defaults."
---

## Phase 1 — Triage

MUST:
- Check SELinux status from admin container: `sheltie` then `getenforce`
- Review audit logs for SELinux denials: `sheltie` then `ausearch -m avc -ts recent`
- Verify dm-verity status: check console output for dm-verity errors

SHOULD:
- Check kernel security settings: `sheltie` then `sysctl -a | grep -E 'kernel\.(kptr_restrict|dmesg_restrict|unprivileged_bpf)'`
- Verify no unauthorized processes are running: `sheltie` then `ps aux`

MAY:
- Run compliance checks against CIS Bottlerocket Benchmark
- Review container security contexts for privileged containers

## Phase 2 — Remediate

MUST:
- For SELinux denials, adjust container security contexts rather than disabling SELinux
- Ensure workloads use appropriate SELinux labels
- Verify kernel lockdown settings are not bypassed

SHOULD:
- Configure kernel sysctl settings via API: `apiclient set kernel.sysctl.net.core.somaxconn="1024"`
- Use Bottlerocket's built-in FIPS support if required: verify FIPS-enabled AMI variant

MAY:
- Implement custom SELinux policies for specialized workloads (requires custom build)

## Safety Ratings

```
safety_ratings:
  - "getenforce / ausearch: GREEN — read-only security status checks"
  - "sysctl -a: GREEN — read-only kernel parameter enumeration"
  - "apiclient set kernel.sysctl: YELLOW — modifies kernel parameters"
  - "Disabling security features: RED — reduces security posture significantly"
```

## Escalation Conditions

- SELinux denials block critical production workloads with no workaround
- dm-verity errors indicate root filesystem tampering or corruption
- Compliance audit requires settings not configurable through the API
- FIPS mode is required but FIPS AMI variant is not available in the region

## Data Sensitivity

- SELinux audit logs: MEDIUM — reveal process names and file access patterns
- Kernel sysctl values: LOW — system configuration parameters
- Process listing: MEDIUM — reveals running workloads and their arguments
- Compliance scan results: HIGH — reveals security posture and vulnerabilities

## Prohibited Actions

- NEVER disable SELinux enforcing mode in production
- NEVER disable dm-verity or modify the root filesystem integrity checks
- NEVER run containers in privileged mode as a workaround for SELinux denials
- NEVER expose kernel debug interfaces or disable kernel lockdown

## Phase 3 — Rollback

1. Revert kernel sysctl changes: `apiclient set kernel.sysctl.<key>=<original-value>`
2. If security contexts were modified, revert pod/task definitions to previous versions
3. If a custom AMI was deployed, roll back to the standard Bottlerocket AMI
4. Reboot to restore default security settings: `apiclient reboot`
5. Re-run compliance scan to verify security posture is restored

## Common Issues

- symptoms: "Container workload fails with 'permission denied' but runs fine on Amazon Linux"
  diagnosis: "SELinux enforcing mode is blocking container file access."
  resolution: "Check ausearch for AVC denials. Adjust container securityContext or use appropriate SELinux labels."

- symptoms: "Kernel sysctl settings cannot be changed via /proc/sys"
  diagnosis: "Bottlerocket's read-only filesystem and kernel lockdown prevent direct sysctl writes."
  resolution: "Use apiclient set kernel.sysctl.<key>=<value> for supported sysctl parameters."

## Output Format

```yaml
root_cause: "security — <specific_cause>"
evidence:
  - type: security_finding
    content: "<finding details>"
severity: HIGH
mitigation:
  immediate: "Adjust security contexts or kernel settings"
  long_term: "Document security exceptions and update compliance baseline"
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
