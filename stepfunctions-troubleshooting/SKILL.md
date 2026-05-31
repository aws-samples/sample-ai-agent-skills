---
name: stepfunctions-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Step Functions
  problems by analyzing execution failures, input/output processing,
  error handling, service integrations, Map state issues, Express
  workflow logging, callback patterns, performance bottlenecks, and
  deployment errors. Activate when: execution failures or timeouts,
  task state errors, permission denied on service calls, input/output
  path processing issues, payload size exceeded, Catch/Retry not
  working as expected, Lambda/ECS/DynamoDB/API Gateway integration
  failures, Map state concurrency or item processing problems,
  distributed Map issues, Express workflow logging gaps, synchronous
  vs asynchronous Express invocation confusion, task token callback
  failures, heartbeat timeouts, execution history event limit,
  throttling, state machine definition errors, version/alias
  deployment issues, or the user says something is wrong with Step
  Functions without naming specific symptoms.
compatibility: >
  Requires AWS CLI with states, cloudwatch, cloudtrail, logs, lambda,
  ecs, dynamodb, and iam permissions. Optional: AWS X-Ray for
  distributed tracing, CloudWatch Logs Insights for Express workflow
  log analysis.
---

# Step Functions Diagnostics

## When to use

Any Step Functions investigation where the console alone is insufficient — execution failures, task timeouts, input/output data flow issues, error handling configuration, service integration problems, Map state concurrency, Express workflow debugging, callback pattern troubleshooting, or deployment errors.

## Investigation workflow

### Step 1 — Collect and triage

```
# List recent executions for a state machine
aws stepfunctions list-executions --state-machine-arn <state-machine-arn> \
  --status-filter FAILED --max-results 10

# Describe a specific execution
aws stepfunctions describe-execution --execution-arn <execution-arn>

# Get execution history (Standard workflows only)
aws stepfunctions get-execution-history --execution-arn <execution-arn> \
  --max-results 100 --reverse-order

# Describe the state machine definition
aws stepfunctions describe-state-machine --state-machine-arn <state-machine-arn>

# Check CloudWatch metrics for the state machine
aws cloudwatch get-metric-statistics --namespace AWS/States \
  --metric-name ExecutionsFailed --dimensions Name=StateMachineArn,Value=<state-machine-arn> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum

# Check throttling
aws cloudwatch get-metric-statistics --namespace AWS/States \
  --metric-name ExecutionThrottled --dimensions Name=StateMachineArn,Value=<state-machine-arn> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum
```

Triage returns:
- Execution status, start/stop times, input/output
- Execution history events showing state transitions and errors
- State machine definition (ASL JSON)
- Execution failure and throttling metrics

If the execution status is FAILED or TIMED_OUT, the execution history IS the primary evidence source. Don't guess — read the history events.

### Step 2 — Domain deep dive (only if needed)

```
# Check IAM role permissions for the state machine
aws iam get-role --role-name <state-machine-role-name>
aws iam list-attached-role-policies --role-name <state-machine-role-name>
aws iam list-role-policies --role-name <state-machine-role-name>

# Simulate permissions for a specific service action
aws iam simulate-principal-policy --policy-source-arn <role-arn> \
  --action-names lambda:InvokeFunction ecs:RunTask dynamodb:PutItem

# CloudTrail for Step Functions API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=states.amazonaws.com --max-results 20

# Check Express workflow logs (CloudWatch Logs)
aws logs filter-log-events --log-group-name /aws/vendedlogs/states/<state-machine-name> \
  --start-time $(date -u -d '1 hour ago' +%s)000 --limit 50

# Check X-Ray traces (if tracing enabled)
aws xray get-trace-summaries --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --filter-expression 'service("AWS::StepFunctions")'
```

Read `references/stepfunctions-guardrails.md` before concluding on any Step Functions issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# Get full execution history (up to 25,000 events)
aws stepfunctions get-execution-history --execution-arn <execution-arn> --max-results 1000

# Check for Map state child executions (distributed Map)
aws stepfunctions list-map-runs --execution-arn <execution-arn>
aws stepfunctions describe-map-run --map-run-arn <map-run-arn>

# Check Lambda function configuration (if Lambda integration)
aws lambda get-function-configuration --function-name <function-name>

# Check ECS task status (if ECS integration)
aws ecs describe-tasks --cluster <cluster> --tasks <task-arn>

# Check DynamoDB table status (if DynamoDB integration)
aws dynamodb describe-table --table-name <table-name>

# Verify state machine definition for syntax issues
aws stepfunctions validate-state-machine-definition --definition file://definition.json --type STANDARD
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws stepfunctions list-executions` | List executions by status (RUNNING, SUCCEEDED, FAILED, TIMED_OUT, ABORTED) |
| `aws stepfunctions describe-execution` | Execution details: status, input, output, start/stop times |
| `aws stepfunctions get-execution-history` | Full event history for Standard workflow executions |
| `aws stepfunctions describe-state-machine` | State machine definition, role ARN, type, logging config |
| `aws stepfunctions list-map-runs` | List distributed Map runs within an execution |
| `aws stepfunctions describe-map-run` | Distributed Map run details: item counts, failures, status |
| `aws stepfunctions validate-state-machine-definition` | Validate ASL definition for syntax errors |
| `aws cloudwatch get-metric-statistics` | Step Functions metrics: ExecutionsFailed, ExecutionThrottled, ExecutionTime |
| `aws cloudtrail lookup-events` | Step Functions API call history and errors |
| `aws logs filter-log-events` | Express workflow execution logs in CloudWatch |
| `aws iam simulate-principal-policy` | Verify state machine role permissions for service calls |
| `aws xray get-trace-summaries` | Distributed tracing for execution flow analysis |

## Gotchas: Step Functions

These are the mistakes commonly made during Step Functions troubleshooting.

- Standard vs Express workflows have fundamentally different characteristics. Standard: up to 1 year duration, exactly-once execution, execution history API available, priced per state transition. Express: up to 5 minutes duration, at-least-once (async) or at-most-once (sync), NO execution history API — use CloudWatch Logs instead, priced per execution and duration.
- Execution history has a hard limit of 25,000 events per execution. Long-running workflows with many iterations (e.g., large Map states, deep loops) can hit this limit, causing the execution to fail with `ExecutionHistoryLimitExceeded`. Use nested workflows (child state machines) to stay under the limit.
- The task token callback pattern (`"Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken"`) pauses execution until `SendTaskSuccess` or `SendTaskFailure` is called with the token. If the token is lost or the external system never calls back, the execution hangs indefinitely unless a `HeartbeatSeconds` or `TimeoutSeconds` is configured.
- Map state concurrency is controlled by `MaxConcurrency`. Default is 0 (unlimited), which can overwhelm downstream services. Always set an explicit `MaxConcurrency` for production workloads to avoid throttling Lambda, DynamoDB, or other integrated services.
- Input/output processing in Step Functions uses five filters applied in order: InputPath → Parameters → (task execution) → ResultSelector → ResultPath → OutputPath. Misunderstanding this pipeline is the #1 cause of "missing data" bugs. Each filter can drop or reshape data.
- Error handling uses Catch and Retry fields on task states. Retry is attempted BEFORE Catch. Error names are matched in order — put specific errors before `States.ALL`. The `States.TaskFailed` error catches all task failures but NOT state machine-level errors like `States.Timeout`.
- Service integration patterns: `.sync` (wait for completion), `.waitForTaskToken` (callback), and default (request-response, fire-and-forget). Using the wrong pattern is a common mistake — e.g., calling Lambda without `.sync` means Step Functions won't wait for the Lambda to finish.
- `HeartbeatSeconds` must be less than `TimeoutSeconds`. HeartbeatSeconds is for long-running tasks that periodically report progress. If the task doesn't send a heartbeat within the interval, the state fails with `States.Timeout`. This is different from the overall task timeout.
- Express workflows have NO execution history API (`get-execution-history` returns nothing). You MUST enable CloudWatch Logs logging on the state machine and query logs for debugging. Without logging enabled, Express workflow failures are invisible.
- State machine definition size limit is 1 MB (1,048,576 bytes). Large definitions with many states or embedded JSON payloads can hit this limit. Use S3 references or external configuration instead of embedding large payloads in the definition.
- Execution name uniqueness: Standard workflows require unique execution names within a 90-day window. Reusing a name within 90 days returns the previous execution result (idempotent). Express workflows have no execution name uniqueness constraint.
- Map state item batcher groups input items into sub-arrays for batch processing. The `MaxItemsPerBatch` and `MaxInputBytesPerBatch` settings control batch sizes. Misconfiguring these can lead to oversized payloads or inefficient processing.
- Distributed Map (`ItemProcessor` with `DISTRIBUTED` mode) processes items in parallel using child workflow executions. It supports reading items from S3 (JSON, CSV, S3 inventory). Unlike inline Map, distributed Map can handle millions of items but has different error handling and concurrency semantics.

## Anti-hallucination rules

1. Always cite specific execution history events, CloudWatch metrics, CloudWatch Logs entries, or CloudTrail events as evidence. Never diagnose from symptoms alone.
2. Never suggest using `get-execution-history` for Express workflows — Express workflows do not support the execution history API. Use CloudWatch Logs instead.
3. Never claim that Step Functions retries are unlimited — Retry has a `MaxAttempts` field (default 3) and `BackoffRate` (default 2.0). After max retries, the error falls through to Catch.
4. Never suggest modifying a running execution's definition — executions use the definition that was active at start time. Changes require a new execution.
5. Never assume service integration patterns are interchangeable — `.sync`, `.waitForTaskToken`, and default (request-response) have fundamentally different behaviors and IAM permission requirements.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Execution Failures | A1–A4 | Task state failures, execution timeouts, permission errors, state machine definition errors |
| B — Input/Output | B1–B3 | Path processing errors, payload size limits, ResultPath confusion |
| C — Error Handling | C1–C3 | Catch/Retry configuration, error matching, fallback states |
| D — Service Integration | D1–D4 | Lambda invoke, ECS RunTask, DynamoDB, API Gateway |
| E — Map State | E1–E3 | Concurrency issues, item processing failures, distributed Map |
| F — Express Workflows | F1–F2 | Logging, synchronous vs async invocation |
| G — Callback Pattern | G1–G2 | Task token issues, heartbeat timeout |
| H — Performance | H1–H2 | Execution history limits, throttling |
| I — Deployment | I1–I2 | Definition errors, version/alias issues |
| Z — Catch-All | Z1 | General Step Functions troubleshooting |
