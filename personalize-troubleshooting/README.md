# Amazon Personalize Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Personalize problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Personalize — dataset management, solution training, campaign deployment, recommendation quality, batch inference, event tracking, and filter configuration.

### Activate When

- Dataset import failures
- Solution training errors or slow training
- Campaign deployment issues
- Poor recommendation quality
- Batch inference failures
- Event tracking problems
- Filter configuration issues
- Cold start problems
- General Personalize errors

---

## Skill Structure

```
personalize-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-dataset-import.md, A2-schema-issues.md, A3-data-quality.md
    ├── B1-solution-training.md, B2-recipe-selection.md
    ├── C1-campaign-deployment.md, C2-campaign-updates.md
    ├── D1-realtime-recommendations.md, D2-recommendation-quality.md
    ├── E1-batch-inference.md, E2-batch-segment.md
    ├── F1-event-tracking.md, F2-event-ingestion.md
    ├── G1-filters.md, H1-domain-recommenders.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `personalize:*`, `personalize-runtime:*`, `personalize-events:*`, `s3:*`, `iam:PassRole`

---

## License

MIT-0
