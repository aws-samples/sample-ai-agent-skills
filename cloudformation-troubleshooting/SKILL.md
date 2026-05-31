---
name: cloudformation-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS CloudFormation and
  CDK problems by analyzing stack events, template errors, drift state,
  resource failures, and following structured runbooks. Activate when:
  CREATE_FAILED, UPDATE_FAILED, DELETE_FAILED, UPDATE_ROLLBACK_FAILED,
  template syntax errors, circular dependencies, parameter issues,
  resource creation failures, dependency failures, resource timeouts,
  resource already exists, drift detection, drift remediation, resource
  import, nested stack failures, output reference errors, custom resource
  Lambda failures, custom resource timeouts, cleanup on delete failures,
  StackSet operation failures, StackSet instance failures, StackSet
  deployment timeouts, change set failures, replacement detection, CDK
  synth errors, CDK deploy failures, CDK bootstrap issues, or the user
  says something is wrong with CloudFormation or CDK without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with CloudFormation, IAM, Lambda,
  CloudWatch Logs, CloudTrail, and optionally CDK CLI permissions.
---

# CloudFormation Diagnostics

## When to use

Any CloudFormation or CDK investigation where the console alone is insufficient — stack failure analysis, template debugging, drift detection, nested stack troubleshooting, custom resource failures, StackSet operations, change set evaluation, resource import, or CDK synthesis and deployment errors.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudformation describe-stacks --stack-name <stack>
aws cloudformation describe-stack-events --stack-name <stack>
aws cloudformation get-template --stack-name <stack>
aws cloudformation detect-stack-drift --stack-name <stack>
```

### Step 2 — Domain deep dive

```
aws cloudformation list-stack-resources --stack-name <stack>
aws cloudformation describe-stack-resource-drifts --stack-name <stack>
aws cloudformation describe-change-set --stack-name <stack> --change-set-name <changeset>
aws cloudformation list-stack-sets
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudformation.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name /aws/lambda/<custom-resource-function> --start-time <epoch-ms>
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/
```

Read `references/cloudformation-guardrails.md` before concluding on any CloudFormation issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-stacks` | Stack status, outputs, parameters |
| `describe-stack-events` | Failure event timeline (read bottom-up) |
| `get-template` | Current deployed template |
| `validate-template` | Syntax and basic structure validation |
| `detect-stack-drift` | Initiate drift detection |
| `describe-stack-resource-drifts` | Drift details per resource |
| `list-stack-resources` | All resources and their status |
| `describe-change-set` | Preview changes before update |
| `continue-update-rollback` | Fix UPDATE_ROLLBACK_FAILED |
| `list-imports` | Find stacks importing an export |
| `describe-stack-set-operation` | StackSet operation details |
| `list-stack-set-operation-results` | Per-account/region StackSet results |

## Gotchas: CloudFormation

- Stack events show failures in reverse chronological order. Read bottom-up for root cause — the FIRST `*_FAILED` event is the real error; subsequent failures are cascading.
- DELETE_FAILED usually means a resource can't be deleted. Common causes: S3 bucket not empty, ENI still in use by another resource, security group referenced by another resource.
- UPDATE_ROLLBACK_FAILED is the worst state. Requires `ContinueUpdateRollback` with `--resources-to-skip` for resources that cannot be rolled back. This is the only way to recover without deleting the stack.
- Circular dependencies are detected at template validation, not deployment. CloudFormation catches them during `validate-template` or at the start of create/update, before any resources are provisioned.
- Custom resources have a 1-hour timeout by default. The Lambda function must send SUCCESS or FAILED to the presigned S3 URL. If it doesn't respond within the timeout, CloudFormation marks it as failed.
- Resource import requires a deletion policy of Retain. The resource must have `DeletionPolicy: Retain` in the template before it can be imported into a stack.
- Nested stacks fail from the inside out. Check the innermost nested stack first — the root cause is always in the deepest failing stack.
- DependsOn creates implicit ordering but doesn't guarantee resource readiness. A resource may be in CREATE_COMPLETE status but not yet fully functional (e.g., an RDS instance that is created but not yet accepting connections).
- Change sets show what WILL change but don't catch all replacement scenarios. Some replacements are conditional on the actual property values and can't be determined until execution.
- Drift detection doesn't cover all resource types. Only resources that support drift detection are checked. Unsupported resources are silently skipped.
- StackSets have concurrency limits and failure tolerance settings. `MaxConcurrentCount`/`MaxConcurrentPercentage` control parallelism. `FailureToleranceCount`/`FailureTolerancePercentage` control how many failures are acceptable before the operation stops.
- CDK synth errors are local (template generation) vs deploy errors are remote (CloudFormation). Synth errors mean the CDK app code has issues. Deploy errors mean the generated template or the AWS environment has issues.
- Template size limits: 51,200 bytes for direct upload, 460,800 bytes via S3 URL.
- Resource limits: maximum 500 resources per stack. Use nested stacks to exceed this.
- `DeletionPolicy` vs `UpdateReplacePolicy`: `DeletionPolicy` controls what happens when the resource is removed from the template or the stack is deleted. `UpdateReplacePolicy` controls what happens when the resource is replaced during an update.
- Stack policy protects resources from updates. A stack policy can prevent updates to specific resources even if the IAM principal has full CloudFormation permissions.
- Macro transforms (e.g., `AWS::Serverless`, `AWS::Include`) are processed before validation. Errors in macros produce cryptic messages.

### Stack status reference

| Status | Meaning |
|--------|---------|
| CREATE_IN_PROGRESS | Stack creation underway |
| CREATE_FAILED | One or more resources failed to create |
| CREATE_COMPLETE | All resources created successfully |
| ROLLBACK_IN_PROGRESS | Creation failed, rolling back |
| ROLLBACK_COMPLETE | Rollback after failed creation complete |
| UPDATE_IN_PROGRESS | Stack update underway |
| UPDATE_COMPLETE | Update succeeded |
| UPDATE_ROLLBACK_IN_PROGRESS | Update failed, rolling back |
| UPDATE_ROLLBACK_COMPLETE | Rollback after failed update complete |
| UPDATE_ROLLBACK_FAILED | Rollback itself failed — manual intervention needed |
| DELETE_IN_PROGRESS | Stack deletion underway |
| DELETE_FAILED | One or more resources failed to delete |
| DELETE_COMPLETE | Stack deleted |

## Anti-hallucination rules

1. Always cite specific stack events, resource status reasons, or API responses as evidence.
2. Stack events must be read bottom-up. Never diagnose from the top of the event list.
3. Rollback success does NOT mean the original operation succeeded. Always find the original failure.
4. Custom resources have a 1-hour timeout, not instant. Never claim custom resource failures are immediate.
5. Drift detection does not cover all resource types. Never claim full drift coverage.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 32 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Stack Failures | A1–A4 | CREATE_FAILED, UPDATE_FAILED, DELETE_FAILED, UPDATE_ROLLBACK_FAILED |
| B — Template Issues | B1–B3 | Syntax errors, circular dependencies, parameter issues |
| C — Resource Errors | C1–C4 | Resource creation failure, already exists, timeout, dependency failure |
| D — Drift | D1–D3 | Drift detection, drift remediation, resource import |
| E — Nested Stacks | E1–E2 | Nested stack failures, output references |
| F — Custom Resources | F1–F3 | Lambda failures, timeout, cleanup on delete |
| G — StackSets | G1–G3 | Operation failures, instance failures, deployment timeouts |
| H — Change Sets | H1–H2 | Change set failures, replacement detection |
| I — CDK | I1–I3 | Synth errors, deploy failures, bootstrap issues |
| Z — Catch-All | Z1 | General troubleshooting |
