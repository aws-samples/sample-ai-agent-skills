---
title: "B3 — Knowledge Base Retrieval Issues"
description: "Diagnose poor retrieval quality from Bedrock knowledge bases"
status: active
severity: MEDIUM
triggers:
  - "poor retrieval quality"
  - "wrong answers from KB"
  - "retrieval not finding documents"
  - "KB search not working"
owner: devops-agent
objective: "Identify and resolve knowledge base retrieval quality issues"
context: "Retrieval quality depends on chunking strategy, embedding model, vector search configuration, number of results, and the retrieval prompt. Poor results stem from suboptimal chunking, stale data (sync needed), incorrect search parameters, or insufficient data."
---

## Phase 1 — Triage

MUST:
- Test retrieval directly: `aws bedrock-agent-runtime retrieve --knowledge-base-id <kb-id> --retrieval-query '{"text":"<query>"}'`
- Check last sync status and timestamp
- Verify the data source contains relevant documents
- Check retrieval configuration (number of results, search type)

SHOULD:
- Test with different query phrasings
- Check chunking strategy configuration
- Verify embedding model is appropriate for the content
- Review retrieval scores for returned results

MAY:
- Test vector store search directly
- Compare different chunking strategies

## Phase 2 — Remediate

MUST:
- Ensure data is synced (run StartIngestionJob if stale)
- Verify documents contain the expected information
- Adjust number of retrieval results if too few

SHOULD:
- Optimize chunking strategy (size, overlap)
- Use hybrid search (semantic + keyword) if available
- Add metadata filters for more precise retrieval
- Test with the RetrieveAndGenerate API for end-to-end quality

MAY:
- Experiment with different embedding models
- Implement retrieval quality evaluation

## Common Issues

- symptoms: "KB returns irrelevant results"
  diagnosis: "Chunking too large or embedding model not suitable."
  resolution: "Reduce chunk size. Increase chunk overlap. Try different embedding model."

- symptoms: "KB returns no results for valid queries"
  diagnosis: "Data not synced or query doesn't match document content."
  resolution: "Run sync. Verify documents contain relevant content. Try different query phrasing."

## Output Format

```yaml
root_cause: "retrieval_issue — <specific_cause>"
evidence:
  - type: retrieval_results
    content: "<retrieval test results and scores>"
  - type: sync_status
    content: "<last sync status and timestamp>"
severity: MEDIUM
mitigation:
  immediate: "Sync data and adjust retrieval parameters"
  long_term: "Optimize chunking strategy and implement quality evaluation"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure data is synced (run StartIngestionJob if stale) | GREEN |
| Verify documents contain the expected information | GREEN |
| Adjust number of retrieval results if too few | GREEN |
| Optimize chunking strategy (size, overlap) | YELLOW |
| Use hybrid search (semantic + keyword) if available | GREEN |
| Add metadata filters for more precise retrieval | GREEN |
| Test with the RetrieveAndGenerate API for end-to-end quality | GREEN |
| Experiment with different embedding models | YELLOW |
| Implement retrieval quality evaluation | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Retrieval queries and results may expose sensitive document content

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If chunking strategy was changed, revert to the previous chunking configuration and re-sync the data source
2. If embedding model was changed, revert to the previous model and re-index all documents
3. If retrieval parameters were modified, restore previous numberOfResults and search type settings
4. Verify rollback by testing retrieval with known queries and comparing result quality

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
