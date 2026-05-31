# AWS Systems Manager Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Systems Manager when the console alone isn't enough — SSM Agent connectivity, Session Manager access, Run Command execution, Patch Manager compliance, State Manager associations, Parameter Store access and throughput, Automation runbooks, Inventory collection, and Maintenance Window scheduling.

### Activate When

- SSM Agent not connecting or showing ConnectionLost
- Managed instance not appearing in Fleet Manager
- Hybrid activation failures for on-premises servers
- Session Manager session failures or disconnects
- Port forwarding not working through Session Manager
- Run Command failures or timeouts
- Run Command targeting issues (instances not found)
- Patch baseline configuration problems
- Patching failures during maintenance windows
- Patch compliance gaps or missing patches
- State Manager association failures
- Configuration drift detection issues
- Parameter Store access denied errors
- Parameter Store throughput throttling
- Parameter tier confusion (standard vs advanced)
- Automation runbook execution failures
- Approval workflow stuck or not triggering
- Inventory collection not working
- Custom inventory schema issues
- Maintenance window not executing
- Maintenance window task failures

---

## Skill Structure

```
ssm-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-agent-not-connecting.md
    ├── A2-managed-instance-registration.md
    ├── A3-hybrid-activation.md
    ├── B1-session-failures.md
    ├── B2-port-forwarding.md
    ├── C1-command-failures.md
    ├── C2-command-timeout.md
    ├── C3-targeting-issues.md
    ├── D1-patch-baseline.md
    ├── D2-patching-failures.md
    ├── D3-patch-compliance.md
    ├── E1-association-failures.md
    ├── E2-drift-detection.md
    ├── F1-parameter-access.md
    ├── F2-parameter-throughput.md
    ├── F3-parameter-tiers.md
    ├── G1-runbook-failures.md
    ├── G2-approval-workflows.md
    ├── H1-collection-failures.md
    ├── H2-custom-inventory.md
    ├── I1-window-execution.md
    ├── I2-task-failures.md
    ├── Z1-general-troubleshooting.md
    ├── ssm-guardrails.md
    └── ssm-hallucination-patterns.yaml
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Agent** | A1–A3 | SSM Agent not connecting, managed instance registration, hybrid activation |
| **B — Session Manager** | B1–B2 | Session failures, port forwarding |
| **C — Run Command** | C1–C3 | Command failures, timeout, targeting |
| **D — Patch Manager** | D1–D3 | Patch baseline, patching failures, compliance |
| **E — State Manager** | E1–E2 | Association failures, drift |
| **F — Parameter Store** | F1–F3 | Parameter access, throughput, tiers |
| **G — Automation** | G1–G2 | Runbook failures, approval workflows |
| **H — Inventory** | H1–H2 | Collection failures, custom inventory |
| **I — Maintenance Windows** | I1–I2 | Window execution, task failures |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/ssm-guardrails.md` covering SSM Agent IAM requirements, VPC endpoint connectivity, Session Manager prerequisites, Run Command timeout behavior, Parameter Store tier limits, patch baseline OS specificity, Automation document schema versions, State Manager association scheduling, Maintenance Window cutoff behavior, hybrid activation requirements, inventory collection intervals, and SSM document version management.

---

## Investigation Workflow

1. **Triage** — Check instance information, ping status, agent version, IAM role, VPC connectivity
2. **Deep Dive** — Examine commands, sessions, patches, associations, parameters, automations
3. **Detailed** — Get command invocations, patch compliance, automation steps, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `ssm:*`, `ec2:DescribeInstances`, `iam:GetRole`, `iam:GetInstanceProfile`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`
- SSM Agent installed on target instances (pre-installed on Amazon Linux 2, Windows Server 2016+)
- IAM instance profile with `AmazonSSMManagedInstanceCore` policy on managed instances

---

## Usage Examples

```
# Check all managed instances
aws ssm describe-instance-information

# Check specific instance connectivity
aws ssm get-connection-status --target i-1234567890abcdef0

# List failed commands
aws ssm list-commands --filters key=Status,value=Failed

# Get command output
aws ssm get-command-invocation --command-id <cmd-id> --instance-id <instance-id>

# Check patch compliance
aws ssm describe-patch-group-state --patch-group "production"

# List parameters
aws ssm describe-parameters --parameter-filters Key=Name,Option=Contains,Values=prod

# Check automation executions
aws ssm describe-automation-executions --filters Key=ExecutionStatus,Values=Failed
```

---

## License

MIT-0
