# Amazon Bedrock Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Bedrock issue.

## Guardrail 1: Model Access Requires Explicit Approval
Each foundation model in Bedrock requires separate access approval through the console. Access is per-region — approving in us-east-1 does not grant access in us-west-2. Without approval, InvokeModel returns AccessDeniedException. Check model access status in the Bedrock console under Model access.

## Guardrail 2: On-Demand Throughput Has Per-Model Rate Limits
Each model has different tokens-per-minute (TPM) and requests-per-minute (RPM) limits for on-demand usage. Exceeding limits returns ThrottlingException with a Retry-After header. Limits vary by model and region. Provisioned throughput provides dedicated capacity with guaranteed rates. Check CloudWatch Throttles metric.

## Guardrail 3: Knowledge Base Sync Is Asynchronous
After creating a knowledge base or updating data sources, you must explicitly trigger a sync via StartIngestionJob. Sync processes documents, generates embeddings, and stores them in the vector database. This can take minutes to hours. Queries return stale or empty results until sync completes. Check sync status with GetIngestionJob.

## Guardrail 4: Agent Orchestration Uses Multi-Step Reasoning
Agents break down user requests into steps, decide which action groups to invoke, process results, and synthesize responses. The orchestration can loop if the agent cannot determine the next step or if action group responses are ambiguous. MaxIterations and idleSessionTTL control orchestration behavior. Excessive loops indicate prompt or action group configuration issues.

## Guardrail 5: Bedrock Guardrails Evaluate Both Input AND Output
Content filters, topic denials, word filters, and sensitive information filters apply to BOTH user input and model output. A guardrail can block the user's question (input) or the model's response (output). Overly restrictive guardrails cause false positives. Test guardrails with representative inputs before production deployment.

## Guardrail 6: Custom Model Training Data Must Follow Specific Format
Training data for fine-tuning must be in JSONL format with model-specific schema. Each line is a JSON object with prompt-completion pairs. Data must be in S3 with proper IAM access. Training can take hours and incurs compute charges. Failed jobs often stem from data format errors, insufficient data, or IAM permission issues.

## Guardrail 7: Provisioned Throughput Is Billed Even When Idle
Provisioned model throughput provides dedicated capacity with guaranteed token rates. However, charges apply regardless of actual usage. Commitment terms (no commitment, 1-month, 6-month) affect pricing. There is no auto-scaling — you get exactly the provisioned capacity. Delete unused provisioned throughput to stop charges.

## Guardrail 8: VPC Endpoints Required for Private Subnet Access
Bedrock API calls from private subnets (no internet access) require VPC interface endpoints. You may need endpoints for both bedrock (control plane) and bedrock-runtime (data plane). Security groups on the endpoints must allow HTTPS (443) from client resources. Private DNS must be enabled on the VPC.

## Guardrail 9: Streaming Responses Use Event Stream Protocol
InvokeModelWithResponseStream returns an event stream with chunks. Clients must handle the AWS event stream protocol. Each chunk contains a partial response. Connection timeouts must account for full response generation. Not all SDKs handle streaming identically — check SDK-specific documentation.

## Guardrail 10: Model Parameters Are Model-Specific
Each foundation model has different request/response formats, parameter names, and supported features. Claude uses anthropic_version and messages format. Titan uses inputText. Llama uses prompt. Using wrong parameters returns ValidationException. Always check model-specific documentation.

## Guardrail 11: Knowledge Base Vector Store Must Be Pre-Configured
Knowledge bases require a vector store (OpenSearch Serverless, Pinecone, Redis Enterprise Cloud, or Aurora PostgreSQL). The vector store must be created and configured BEFORE creating the knowledge base. The vector index/collection must have the correct dimensions matching the embedding model. Misconfigured vector stores cause sync failures.

## Guardrail 12: Agent Versions Are Immutable After Creation
Agent aliases point to specific agent versions. Creating a new version snapshots the current configuration. Published versions cannot be modified — create a new version for changes. The DRAFT version is mutable and used for testing. Always test with DRAFT before publishing a new version.
