# Amazon Forecast Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Forecast problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Forecast — dataset management, predictor training, forecast generation, accuracy evaluation, what-if analysis, and explainability.

### Activate When

- Dataset import failures
- Predictor training errors
- Forecast generation issues
- Poor forecast accuracy
- What-if analysis problems
- Explainability errors
- General Forecast errors

---

## Skill Structure

```
forecast-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-dataset-import.md, A2-data-format.md
    ├── B1-predictor-training.md, B2-auto-predictor.md, B3-accuracy-metrics.md
    ├── C1-forecast-generation.md, C2-forecast-query.md
    ├── D1-whatif-analysis.md, D2-whatif-forecast.md
    ├── E1-explainability.md
    ├── F1-timeseries-quality.md, F2-missing-values.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `forecast:*`, `forecastquery:*`, `s3:*`, `iam:PassRole`

---

## License

MIT-0
