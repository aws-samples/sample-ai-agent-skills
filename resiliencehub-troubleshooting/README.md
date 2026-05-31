# AWS Resilience Hub Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Resilience Hub problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Resilience Hub — application creation and import, resiliency policy configuration, assessment failures, recommendation interpretation, drift detection, and SOP generation.

### Activate When

- Application import or creation failures
- Resource discovery issues
- Resiliency policy configuration errors
- Assessment failures or unexpected results
- Component compliance issues
- Drift detection not working
- SOP generation errors
- General Resilience Hub errors

---

## Skill Structure

```
resiliencehub-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-application-creation.md
    ├── A2-application-import.md
    ├── A3-resource-discovery.md
    ├── B1-policy-configuration.md
    ├── B2-policy-assignment.md
    ├── C1-assessment-failures.md
    ├── C2-compliance-status.md
    ├── C3-component-analysis.md
    ├── D1-architecture-recommendations.md
    ├── D2-cost-optimization.md
    ├── E1-drift-detection.md
    ├── E2-drift-resolution.md
    ├── F1-sop-generation.md
    ├── F2-sop-execution.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Application** | A1–A3 | Creation, import, resource discovery |
| **B — Policy** | B1–B2 | Configuration, assignment |
| **C — Assessment** | C1–C3 | Failures, compliance, components |
| **D — Recommendations** | D1–D2 | Architecture, cost optimization |
| **E — Drift** | E1–E2 | Detection, resolution |
| **F — SOP** | F1–F2 | Generation, execution |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `resiliencehub:*`, `cloudformation:Describe*`, `resource-groups:*`, `ssm:*` permissions

---

## License

MIT-0
