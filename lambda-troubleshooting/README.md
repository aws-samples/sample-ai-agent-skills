# Lambda Function Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Lambda function problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Lambda functions when CloudWatch metrics alone aren't enough — invocation errors, timeout analysis, cold start optimization, concurrency throttling, VPC networking, IAM permissions, deployment issues, event source mapping failures, and performance tuning.

### Activate When

- Function is timing out or returning errors
- Cold starts are too slow
- Concurrency is being throttled
- Function can't reach VPC resources or the internet
- Memory/CPU is insufficient for the workload
- Deployment fails (package too large, layer issues)
- Permissions are denied (execution role, resource policy, KMS)
- Event source mappings fail (SQS, Kinesis, DynamoDB Streams)
- Logs are missing or X-Ray traces show issues
- OOM kills or runtime crashes
- The user says something is wrong with a Lambda function

---

## Skill Structure

```
lambda-troubleshooting/
├── SKILL.md                          # Main skill definition and investigation workflow
├── README.md                         # This file
└── references/
    ├── A1-timeout.md
    ├── A2-out-of-memory.md
    ├── A3-unhandled-exception.md
    ├── A4-runtime-crash.md
    ├── B1-cold-start-latency.md
    ├── B2-vpc-cold-start.md
    ├── B3-provisioned-concurrency.md
    ├── C1-throttling.md
    ├── C2-reserved-concurrency.md
    ├── C3-account-concurrency-limit.md
    ├── D1-vpc-no-internet.md
    ├── D2-vpc-eni-limits.md
    ├── D3-security-group-blocking.md
    ├── D4-dns-resolution.md
    ├── E1-execution-role.md
    ├── E2-resource-policy.md
    ├── E3-kms-permissions.md
    ├── E4-cross-account.md
    ├── F1-package-size.md
    ├── F2-layer-issues.md
    ├── F3-container-image.md
    ├── G1-sqs-event-source.md
    ├── G2-kinesis-dynamodb-streams.md
    ├── G3-api-gateway.md
    ├── G4-s3-event.md
    ├── H1-memory-cpu-tuning.md
    ├── H2-io-bottleneck.md
    ├── H3-sdk-initialization.md
    ├── I1-missing-logs.md
    ├── I2-xray-tracing.md
    ├── Z1-general-troubleshooting.md
    ├── lambda-guardrails.md
    └── lambda-hallucination-patterns.yaml
```

---

## Runbook Library (31 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Invocation Errors** | A1–A4 | Timeout, OOM, unhandled exception, runtime crash |
| **B — Cold Starts** | B1–B3 | Init duration, VPC cold start, provisioned concurrency |
| **C — Concurrency** | C1–C3 | Throttling, reserved concurrency, account limits |
| **D — Networking** | D1–D4 | VPC no internet, ENI limits, security group blocking, DNS |
| **E — Permissions** | E1–E4 | Execution role, resource policy, KMS, cross-account |
| **F — Deployment** | F1–F3 | Package size, layer issues, container image |
| **G — Event Sources** | G1–G4 | SQS, Kinesis/DynamoDB Streams, API Gateway, S3 |
| **H — Performance** | H1–H3 | Memory/CPU tuning, I/O bottleneck, SDK initialization |
| **I — Observability** | I1–I2 | Missing logs, X-Ray tracing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails & Anti-Hallucination

### Lambda Guardrails (`lambda-guardrails.md`)
12 rules covering: memory-CPU proportionality, reserved vs provisioned concurrency, VPC internet access, /tmp ephemerality, timeout limits, sync vs async retry behavior, cold start mitigation, package size limits, environment variable limits, execution role vs resource policy, event source mapping concurrency, and layer compatibility.

### Hallucination Patterns (`lambda-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about Lambda, including:
- Claiming CPU is independent of memory (it's proportional)
- Confusing reserved concurrency with provisioned concurrency
- Claiming VPC Lambda has internet access by default
- Applying async retry behavior to synchronous invocations
- Treating /tmp as reliable persistent storage

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using CloudWatch logs, metrics, and function configuration. Identify the failure domain (invocation error, cold start, throttling, networking, permissions, deployment).

### Phase 2 — Enrich / Remediate
Deep dive using X-Ray traces, VPC configuration, IAM policy analysis, event source mapping state, or layer inspection.

### Phase 3 — Report
State root cause with evidence, severity, and recommended mitigations.

### Output Format
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI or SDK access with Lambda, CloudWatch Logs, CloudWatch Metrics, and IAM permissions
- For VPC debugging: VPC, EC2 subnet/security group permissions
- For tracing: X-Ray permissions (optional)
- For event source debugging: SQS, Kinesis, DynamoDB, S3, API Gateway permissions as needed

---

## Usage Examples

### Function Timing Out
```
Lambda function my-api-handler is timing out. It's in a VPC. Check if
it can reach the internet, look at duration metrics, and check for
downstream service latency.
```

### Throttling Investigation
```
Lambda function my-processor is being throttled. Check reserved
concurrency settings, account-level limits, and whether other
functions are consuming the shared pool.
```

### Cold Start Optimization
```
Lambda function my-api has cold starts over 5 seconds. It uses a
VPC and has a 50MB deployment package. What can we do to reduce
cold start latency?
```

### Deployment Failure
```
Lambda deployment failed with "Unzipped size must be smaller than
262144000 bytes." Check the package size and layers.
```

---

## License

MIT-0
