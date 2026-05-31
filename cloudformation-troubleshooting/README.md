# CloudFormation Diagnostics Skill

Agent skill for investigating and troubleshooting AWS CloudFormation and CDK problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudFormation and CDK when the console alone isn't enough — stack failure analysis, template debugging, drift detection, nested stack troubleshooting, custom resource failures, StackSet operations, change set evaluation, resource import, and CDK synthesis and deployment errors.

### Activate When

- Stack creation failures (CREATE_FAILED, ROLLBACK_COMPLETE)
- Stack update failures (UPDATE_FAILED, UPDATE_ROLLBACK_COMPLETE)
- Stack deletion failures (DELETE_FAILED)
- UPDATE_ROLLBACK_FAILED requiring manual intervention
- Template syntax or validation errors
- Circular dependency errors
- Parameter validation failures
- Individual resource creation failures
- Resource already exists conflicts
- Resource timeout errors
- Resource dependency failures
- Drift detection and remediation
- Resource import issues
- Nested stack failures and output references
- Custom resource Lambda failures and timeouts
- StackSet operation and instance failures
- Change set creation or execution failures
- CDK synth errors, deploy failures, or bootstrap issues

---

## Skill Structure

```
cloudformation-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-create-failed.md
    ├── A2-update-failed.md
    ├── A3-delete-failed.md
    ├── A4-update-rollback-failed.md
    ├── B1-syntax-errors.md
    ├── B2-circular-dependencies.md
    ├── B3-parameter-issues.md
    ├── C1-resource-creation-failure.md
    ├── C2-already-exists.md
    ├── C3-resource-timeout.md
    ├── C4-dependency-failure.md
    ├── D1-drift-detection.md
    ├── D2-drift-remediation.md
    ├── D3-import-resources.md
    ├── E1-nested-stack-failures.md
    ├── E2-output-references.md
    ├── F1-lambda-backed-failures.md
    ├── F2-custom-resource-timeout.md
    ├── F3-cleanup-on-delete.md
    ├── G1-stackset-operation-failures.md
    ├── G2-instance-failures.md
    ├── G3-stackset-deployment-timeouts.md
    ├── H1-change-set-failures.md
    ├── H2-replacement-detection.md
    ├── I1-cdk-synth-errors.md
    ├── I2-cdk-deploy-failures.md
    ├── I3-cdk-bootstrap-issues.md
    ├── Z1-general-troubleshooting.md
    ├── cloudformation-guardrails.md
    └── cloudformation-hallucination-patterns.yaml
```

---

## Runbook Library (32 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Stack Failures** | A1–A4 | CREATE_FAILED, UPDATE_FAILED, DELETE_FAILED, UPDATE_ROLLBACK_FAILED |
| **B — Template Issues** | B1–B3 | Syntax errors, circular dependencies, parameter issues |
| **C — Resource Errors** | C1–C4 | Resource creation failure, already exists, timeout, dependency failure |
| **D — Drift** | D1–D3 | Drift detection, drift remediation, resource import |
| **E — Nested Stacks** | E1–E2 | Nested stack failures, output references |
| **F — Custom Resources** | F1–F3 | Lambda-backed failures, timeout, cleanup on delete |
| **G — StackSets** | G1–G3 | StackSet operation failures, instance failures, deployment timeouts |
| **H — Change Sets** | H1–H2 | Change set failures, replacement detection |
| **I — CDK** | I1–I3 | CDK synth errors, CDK deploy failures, CDK bootstrap issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
