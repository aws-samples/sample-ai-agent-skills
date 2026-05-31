---
name: bedrock-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Bedrock problems
  by analyzing model access, invocation errors, throttling, knowledge base
  creation, sync failures, retrieval issues, agent creation, action group
  errors, orchestration problems, custom model training, provisioned
  throughput, content filtering, topic denial, latency, token limits,
  IAM permissions, VPC configuration, Lambda integration, streaming
  responses, and following structured runbooks. Activate when: model
  access denied, invocation errors, throttling/rate limits, knowledge
  base creation failures, data sync issues, poor retrieval quality,
  agent creation errors, action group failures, orchestration loops,
  fine-tuning failures, provisioned throughput issues, guardrail blocking
  legitimate content, latency problems, token limit exceeded, IAM
  permission errors, VPC endpoint issues, Lambda timeout with Bedrock,
  streaming response errors, or the user says something is wrong with
  Bedrock without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Bedrock, Bedrock Agent, S3, IAM,
  Lambda, CloudWatch, CloudTrail, and optionally OpenSearch Serverless,
  Pinecone, or other vector store permissions. Some operations require
  model access approval.
---

# Amazon Bedrock Diagnostics

## When to use

Any Amazon Bedrock investigation where the console alone is insufficient — model access and invocation issues, knowledge base configuration, agent orchestration, custom model training, guardrails configuration, performance optimization, security setup, or service integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws bedrock list-foundation-models --query 'modelSummaries[*].{Id:modelId,Name:modelName,Provider:providerName,Status:modelLifecycle.status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws bedrock get-model-invocation-logging-configuration
aws bedrock list-custom-models
aws bedrock-agent list-agents --query 'agentSummaries[*].{Id:agentId,Name:agentName,Status:agentStatus}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 2 — Domain deep dive

```
aws bedrock-agent list-knowledge-bases --query 'knowledgeBaseSummaries[*].{Id:knowledgeBaseId,Name:name,Status:status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws bedrock-agent get-knowledge-base --knowledge-base-id <kb-id>
aws bedrock get-guardrail --guardrail-identifier <guardrail-id>
aws bedrock list-provisioned-model-throughputs
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Bedrock --metric-name Invocations --dimensions Name=ModelId,Value=<model-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws bedrock-agent get-agent --agent-id <agent-id>
aws bedrock-agent list-agent-action-groups --agent-id <agent-id> --agent-version DRAFT
```

Read `references/bedrock-guardrails.md` before concluding on any Bedrock issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `bedrock list-foundation-models` | Check available models and status |
| `bedrock invoke-model` | Test model invocation directly |
| `bedrock-agent list-knowledge-bases` | List knowledge bases and status |
| `bedrock-agent get-knowledge-base` | Get KB configuration details |
| `bedrock-agent list-agents` | List agents and status |
| `bedrock-agent get-agent` | Get agent configuration |
| `bedrock get-guardrail` | Check guardrail configuration |
| `bedrock list-provisioned-model-throughputs` | Check provisioned throughput |
| `bedrock list-custom-models` | List fine-tuned models |
| CloudWatch metrics | Monitor invocation counts, latency, errors |

## Gotchas: Amazon Bedrock

- Model access must be explicitly requested and approved. Each foundation model requires separate access approval in the Bedrock console. Access is per-region. Without approval, InvokeModel returns AccessDeniedException.
- On-demand throughput has rate limits per model. Each model has different tokens-per-minute (TPM) and requests-per-minute (RPM) limits. Exceeding limits returns ThrottlingException. Provisioned throughput provides dedicated capacity.
- Knowledge base sync is asynchronous. After creating a KB or updating data sources, you must trigger a sync. Sync can take minutes to hours depending on data volume. Queries return stale results until sync completes.
- Agents use a multi-step orchestration loop. Agents break down tasks into steps, invoke action groups, and synthesize responses. Orchestration can loop if the agent cannot determine the next step. Max iterations and timeout settings control this.
- Guardrails evaluate both input and output. Content filters, topic denials, and word filters apply to both user input and model output. Overly restrictive guardrails block legitimate use cases. Test guardrails thoroughly before production.
- Custom model training requires specific S3 data format. Training data must be in JSONL format with specific schema. Training can take hours. Failed training jobs often stem from data format issues or insufficient IAM permissions.
- Provisioned throughput is billed even when idle. Provisioned model throughput provides dedicated capacity but incurs charges regardless of usage. Commitment terms (1-month, 6-month) affect pricing. No auto-scaling.
- VPC endpoints are required for private access. Bedrock API calls from private subnets require VPC endpoints. Both bedrock and bedrock-runtime endpoints may be needed. Security groups must allow HTTPS traffic.
- Streaming responses require specific client handling. InvokeModelWithResponseStream returns chunks. Client must handle the event stream protocol. Timeouts must account for full response generation time.

## Anti-hallucination rules

1. Always cite specific model IDs, KB IDs, agent IDs, or API responses as evidence.
2. Model access requires explicit approval. Never assume a model is accessible.
3. On-demand has rate limits. Never claim unlimited throughput without provisioned capacity.
4. KB sync is async. Never assume data is immediately available after upload.
5. Guardrails evaluate both input AND output. Never claim they only filter one direction.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Model | A1-A3 | Model access, invocation errors, throttling |
| B — Knowledge Base | B1-B3 | KB creation, sync failures, retrieval issues |
| C — Agents | C1-C3 | Agent creation, action group errors, orchestration |
| D — Fine-Tuning | D1-D2 | Custom model training, provisioned throughput |
| E — Guardrails | E1-E2 | Content filtering, topic denial |
| F — Performance | F1-F2 | Latency, token limits |
| G — Security | G1-G2 | IAM permissions, VPC config |
| H — Integration | H1-H2 | Lambda integration, streaming responses |
| Z — Catch-All | Z1 | General troubleshooting |
