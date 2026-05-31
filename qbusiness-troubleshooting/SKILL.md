---
name: qbusiness-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Q Business problems
  by analyzing application creation, data source connectors, retrieval
  quality, guardrails, plugins, web experience, IAM Identity Center
  integration, and following structured runbooks. Activate when: application
  creation failures, connector sync errors, poor retrieval, guardrail
  blocking, plugin errors, web experience issues, SSO problems, or the
  user says something is wrong with Q Business.
compatibility: >
  Requires AWS CLI with qbusiness, S3, IAM, SSO, CloudWatch, Lambda,
  and Secrets Manager permissions.
---

# Amazon Q Business Diagnostics

## When to use

Any Amazon Q Business investigation — application setup, data source connectors, retrieval quality, guardrails, plugins, web experience, or IAM Identity Center integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws qbusiness list-applications
aws qbusiness get-application --application-id <app-id>
aws qbusiness list-indices --application-id <app-id>
aws qbusiness list-data-sources --application-id <app-id> --index-id <idx-id>
```

### Step 2 — Domain deep dive

```
aws qbusiness list-data-source-sync-jobs --application-id <app-id> --index-id <idx-id> --data-source-id <ds-id>
aws qbusiness list-plugins --application-id <app-id>
aws qbusiness get-web-experience --application-id <app-id> --web-experience-id <we-id>
aws qbusiness list-retrievers --application-id <app-id>
```

### Step 3 — Detailed investigation

```
aws qbusiness get-chat-controls-configuration --application-id <app-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=qbusiness.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Q Business issue.

## Gotchas: Amazon Q Business

- Q Business requires IAM Identity Center. Applications cannot function without SSO.
- Data source sync is asynchronous. Documents are not available until sync completes.
- Guardrails (topic controls) can block legitimate queries. Test thoroughly.
- Plugins extend Q Business with actions. Plugin errors affect conversation flow.
- Web experience requires proper IAM Identity Center user assignment.
- Retrieval quality depends on document quality and chunking strategy.

## Anti-hallucination rules

1. Always cite application IDs, data source IDs, or API responses as evidence.
2. Q Business requires IAM Identity Center. Never suggest it works without SSO.
3. Sync must complete before documents are available for chat.
4. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Application | A1-A2 | App creation, app configuration |
| B — Data Sources | B1-B2 | Connector sync, connector config |
| C — Retrieval | C1-C2 | Retrieval quality, retriever config |
| D — Guardrails | D1-D2 | Topic controls, response controls |
| E — Plugins | E1-E2 | Plugin creation, plugin execution |
| F — Web Experience | F1-F2 | Web experience setup, customization |
| G — Identity | G1-G2 | IAM Identity Center, user mapping |
