# Amazon Bedrock Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Bedrock problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Bedrock when the console alone isn't enough — model access and invocation, knowledge base management, agent orchestration, custom model training, guardrails configuration, performance optimization, security setup, and service integration.

### Activate When

- Model access denied or not available
- Model invocation errors or unexpected responses
- Throttling or rate limit exceeded
- Knowledge base creation or configuration failures
- Data source sync failures or slow sync
- Poor retrieval quality from knowledge bases
- Agent creation or configuration errors
- Action group Lambda failures
- Agent orchestration loops or timeouts
- Custom model training failures
- Provisioned throughput issues
- Guardrails blocking legitimate content
- High latency on model invocations
- Token limit exceeded errors
- IAM permission errors for Bedrock operations
- VPC endpoint configuration issues
- Lambda integration timeouts
- Streaming response handling errors
- General Bedrock errors without clear symptoms

---

## Skill Structure

```
bedrock-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-model-access.md
    ├── A2-invocation-errors.md
    ├── A3-throttling.md
    ├── B1-kb-creation.md
    ├── B2-sync-failures.md
    ├── B3-retrieval-issues.md
    ├── C1-agent-creation.md
    ├── C2-action-group-errors.md
    ├── C3-orchestration.md
    ├── D1-custom-model-training.md
    ├── D2-provisioned-throughput.md
    ├── E1-content-filtering.md
    ├── E2-topic-denial.md
    ├── F1-latency.md
    ├── F2-token-limits.md
    ├── G1-iam-permissions.md
    ├── G2-vpc-config.md
    ├── H1-lambda-integration.md
    ├── H2-streaming-responses.md
    ├── Z1-general-troubleshooting.md
    ├── bedrock-guardrails.md
    └── bedrock-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Model** | A1–A3 | Model access, invocation errors, throttling |
| **B — Knowledge Base** | B1–B3 | KB creation, sync failures, retrieval issues |
| **C — Agents** | C1–C3 | Agent creation, action group errors, orchestration |
| **D — Fine-Tuning** | D1–D2 | Custom model training, provisioned throughput |
| **E — Guardrails** | E1–E2 | Content filtering, topic denial |
| **F — Performance** | F1–F2 | Latency, token limits |
| **G — Security** | G1–G2 | IAM permissions, VPC config |
| **H — Integration** | H1–H2 | Lambda integration, streaming responses |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/bedrock-guardrails.md` covering model access approval, on-demand rate limits, knowledge base sync behavior, agent orchestration model, guardrail bidirectional evaluation, training data format, provisioned throughput billing, VPC endpoint requirements, streaming protocol, model-specific parameters, knowledge base vector store, and agent versioning.

---

## Investigation Workflow

1. **Triage** — Check model access, invocation logs, KB status, agent status
2. **Deep Dive** — Examine KB configuration, agent action groups, guardrail settings
3. **Detailed** — Review CloudTrail events, CloudWatch metrics, application logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `bedrock:*`, `bedrock-agent:*`, `s3:GetObject`, `lambda:InvokeFunction`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Model access approved for target foundation models
- Vector store access for knowledge base troubleshooting

---

## Usage Examples

```
# List available foundation models
aws bedrock list-foundation-models

# Test model invocation
aws bedrock-runtime invoke-model --model-id anthropic.claude-3-sonnet-20240229-v1:0 --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":256,"messages":[{"role":"user","content":"Hello"}]}' output.json

# List knowledge bases
aws bedrock-agent list-knowledge-bases

# List agents
aws bedrock-agent list-agents

# Check guardrail
aws bedrock list-guardrails
```

---

## License

MIT-0
