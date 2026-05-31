# Amazon Lookout for Vision Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Lookout for Vision problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Lookout for Vision — project creation, dataset imports, model training, model evaluation, inference errors, edge deployment, image quality, labeling, S3 permissions, model packaging, and Greengrass integration.

### Activate When

- Project creation failures
- Dataset import errors
- Model training failures
- Model evaluation issues
- Inference errors
- Edge deployment failures
- Image quality issues
- Labeling errors
- S3 permission problems
- Model packaging errors
- Greengrass integration issues
- General Lookout for Vision errors

---

## Skill Structure

```
lookout-vision-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-project-creation-failures.md
    ├── A2-dataset-import-errors.md
    ├── B1-model-training-failures.md
    ├── B2-model-evaluation-issues.md
    ├── C1-inference-errors.md
    ├── C2-edge-deployment-failures.md
    ├── D1-image-quality-issues.md
    ├── D2-labeling-errors.md
    ├── E1-s3-permissions.md
    ├── E2-model-packaging.md
    ├── F1-greengrass-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Project & Data** | A1–A2 | Project creation failures, dataset import errors |
| **B — Training** | B1–B2 | Model training failures, model evaluation issues |
| **C — Inference** | C1–C2 | Inference errors, edge deployment failures |
| **D — Quality** | D1–D2 | Image quality issues, labeling errors |
| **E — Access** | E1–E2 | S3 permissions, model packaging |
| **F — Integration** | F1 | Greengrass integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `lookoutvision:*`, `s3:Get*`, `iam:Get*`, `greengrass:*`, `kms:Describe*` permissions

---

## License

MIT-0
