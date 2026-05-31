# Control Tower Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Control Tower problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Control Tower when the console alone isn't enough — landing zone setup/update failures, account provisioning errors, guardrail (control) compliance violations, drift detection and resolution, SCP conflicts, IAM Identity Center access issues, StackSet deployment failures, and customization pipeline errors.

### Activate When

- Landing zone setup or update fails
- Account provisioning via Account Factory fails
- Controls (guardrails) fail to enable or show non-compliant
- Drift is detected on landing zone, OUs, or accounts
- Service Catalog provisioned products are stuck or errored
- SSO / IAM Identity Center access issues
- CloudTrail organization trail or Config aggregation is broken
- Nested OU registration or control issues
- Account enrollment or import fails
- Customizations for Control Tower (CfCT) pipeline errors
- SCP conflicts block legitimate operations
- StackSet deployments fail across member accounts
- The user says something is wrong with Control Tower

---

## Skill Structure

```
control-tower-troubleshooting/
├── SKILL.md                          # Main skill definition and investigation workflow
├── README.md                         # This file
└── references/
    ├── A1-landing-zone-setup-failure.md
    ├── A2-landing-zone-update-failure.md
    ├── A3-landing-zone-version-mismatch.md
    ├── A4-region-configuration.md
    ├── B1-account-factory-failure.md
    ├── B2-email-conflicts.md
    ├── B3-account-enrollment-failure.md
    ├── B4-account-import.md
    ├── C1-control-enable-failure.md
    ├── C2-non-compliant-resources.md
    ├── C3-scp-conflicts.md
    ├── C4-proactive-control-issues.md
    ├── D1-landing-zone-drift.md
    ├── D2-ou-drift.md
    ├── D3-account-drift.md
    ├── D4-scp-drift.md
    ├── E1-ou-registration-failure.md
    ├── E2-nested-ou-issues.md
    ├── E3-ou-re-registration.md
    ├── F1-sso-configuration-issues.md
    ├── F2-permission-set-failures.md
    ├── F3-sso-access-problems.md
    ├── G1-cloudtrail-issues.md
    ├── G2-config-aggregation.md
    ├── G3-log-archive-problems.md
    ├── H1-cfct-pipeline-failures.md
    ├── H2-lifecycle-event-issues.md
    ├── H3-custom-scp-conflicts.md
    ├── I1-stackset-failures.md
    ├── I2-stack-instance-drift.md
    ├── I3-deployment-timeouts.md
    ├── Z1-general-troubleshooting.md
    ├── control-tower-guardrails.md
    └── control-tower-hallucination-patterns.yaml
```

---

## Runbook Library (34 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Landing Zone** | A1–A4 | Setup failure, update failure, version mismatch, region configuration |
| **B — Account Provisioning** | B1–B4 | Account Factory failure, email conflicts, enrollment failure, account import |
| **C — Controls (Guardrails)** | C1–C4 | Enable failure, non-compliant resources, SCP conflicts, proactive control issues |
| **D — Drift** | D1–D4 | Landing zone drift, OU drift, account drift, SCP drift |
| **E — Organizational Units** | E1–E3 | OU registration failure, nested OU issues, OU re-registration |
| **F — IAM Identity Center (SSO)** | F1–F3 | SSO configuration issues, permission set failures, access problems |
| **G — Logging & Monitoring** | G1–G3 | CloudTrail issues, Config aggregation, log archive problems |
| **H — Customizations** | H1–H3 | CfCT pipeline failures, lifecycle event issues, custom SCP conflicts |
| **I — StackSet Operations** | I1–I3 | StackSet failures, stack instance drift, deployment timeouts |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails & Anti-Hallucination

### Control Tower Guardrails (`control-tower-guardrails.md`)
12 rules covering: SCP management account exemption, control inheritance (none), detective vs preventive vs proactive controls, CT-managed resource modification, landing zone update prerequisites, Account Factory Service Catalog dependency, drift resolution ordering, OU registration requirements, IAM role dependencies, nested OU control behavior, region deny guardrail effects, and CloudTrail organization trail behavior.

### Hallucination Patterns (`control-tower-hallucination-patterns.yaml`)
10 patterns that LLMs commonly get wrong about Control Tower, including:
- Claiming SCPs apply to the management account (they don't)
- Claiming controls are inherited by child OUs (they aren't)
- Confusing detective controls with preventive controls
- Claiming drift can be ignored safely
- Recommending direct edits to CT-managed CloudFormation stacks

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using Control Tower APIs, CloudFormation stack events, Service Catalog provisioned product status, and Organizations state. Classify the failure domain.

### Phase 2 — Enrich / Remediate
Deep dive into the specific domain using StackSet instance details, Config compliance, SCP analysis, IAM Identity Center configuration, or CloudTrail events.

### Phase 3 — Report
State root cause with evidence, blast radius, severity, and recommended mitigations (immediate and long-term).

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI or SDK access in the management account with Control Tower, Organizations, Service Catalog, CloudFormation, Config, CloudTrail, IAM Identity Center, and STS permissions
- For account provisioning debugging: Service Catalog and CloudFormation permissions
- For compliance analysis: Config and Organizations permissions
- For SSO debugging: IAM Identity Center (sso-admin) permissions

---

## Usage Examples

### Landing Zone Update Failure
```
Control Tower landing zone update to version 3.3 failed. Check the
landing zone operation status, CloudFormation StackSet failures, and
whether drift needs to be resolved first.
```

### Account Provisioning Failure
```
Account Factory failed to provision a new account. The Service Catalog
provisioned product is in ERROR state. Check the CloudFormation stack
events in the management account and the provisioned product record.
```

### Guardrail Non-Compliance
```
Several accounts are showing non-compliant for the detective control
AWS-GR_ENCRYPTED_VOLUMES. Check which accounts and resources are
non-compliant and what remediation is needed.
```

### Drift Detection
```
Control Tower is showing drift detected. Identify what drifted (OU,
account, SCP, or landing zone), what changed, and how to resolve it.
```

### General Triage
```
Something is wrong with our Control Tower setup but I'm not sure what.
Run a general investigation — check landing zone status, drift, recent
operations, and follow whichever runbook matches the symptoms.
```

---

## License

MIT-0
