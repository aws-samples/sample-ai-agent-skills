# AWS CloudFormation Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CloudFormation issue.

## Guardrail 1: Stack Events Are Reverse Chronological — Read Bottom-Up
Stack events are listed newest-first. The FIRST failure event (at the bottom) is the root cause. Subsequent failures are cascading effects from the original error. Never diagnose from the top of the event list. Always scroll to the first `*_FAILED` event and read the `ResourceStatusReason` field.

## Guardrail 2: Rollback Success Does NOT Mean the Operation Succeeded
When a stack rolls back successfully (ROLLBACK_COMPLETE or UPDATE_ROLLBACK_COMPLETE), it means the rollback worked — NOT that the original create/update succeeded. The original failure event is pushed down by rollback events. Always find the original `*_FAILED` event before the rollback began.

## Guardrail 3: UPDATE_ROLLBACK_FAILED Requires Manual Intervention
This is the worst stack state. The only recovery options are: (1) `continue-update-rollback` with `--resources-to-skip` to skip resources that cannot be rolled back, or (2) delete the stack entirely. There is no automatic recovery. The `--resources-to-skip` parameter accepts logical resource IDs of resources that CloudFormation should skip during rollback.

## Guardrail 4: Custom Resources Have a 1-Hour Timeout
Lambda-backed custom resources must send a SUCCESS or FAILED response to the pre-signed S3 URL within 1 hour (default). If the Lambda function crashes, times out, or fails to send the response, CloudFormation waits the full hour before marking the resource as failed. Check Lambda logs first — the function may have completed but failed to send the response.

## Guardrail 5: Circular Dependencies Are Detected at Validation Time
CloudFormation detects circular dependencies during template validation or at the start of create/update, before any resources are provisioned. If you see a circular dependency error, it is a template design issue. Restructure using `Fn::ImportValue`, `Fn::Sub`, or by breaking the cycle with an intermediate resource.

## Guardrail 6: Drift Detection Does NOT Cover All Resource Types
Only resources that support drift detection are checked. Unsupported resources are silently skipped — they will NOT appear in drift results. Check the AWS documentation for the list of supported resource types. Never claim a resource is not drifted just because it doesn't appear in drift results.

## Guardrail 7: DependsOn Does NOT Guarantee Resource Readiness
`DependsOn` ensures CloudFormation waits for the dependency to reach CREATE_COMPLETE before starting the dependent resource. However, CREATE_COMPLETE does not mean the resource is fully functional. For example, an RDS instance may be CREATE_COMPLETE but still initializing. Use wait conditions or custom resources for readiness checks.

## Guardrail 8: Change Sets Preview but Don't Guarantee Success
A change set can be created successfully but fail during execution. Change sets show what CloudFormation plans to do, but runtime conditions (IAM permissions, resource limits, service quotas) can cause execution failures. Some replacement scenarios are conditional and cannot be determined until execution.

## Guardrail 9: Nested Stacks Fail from the Inside Out
When a nested stack fails, the root cause is in the innermost nested stack. The parent stack event will say "Embedded stack <arn> was not successfully created/updated." Always navigate to the nested stack and check its events. Repeat for deeply nested stacks until you find the actual resource failure.

## Guardrail 10: Resource Import Requires DeletionPolicy Retain
Before importing a resource into a stack, the template must declare the resource with `DeletionPolicy: Retain`. The resource must already exist in AWS and must not be managed by another stack. The import operation matches the resource by its identifier (e.g., physical resource ID).

## Guardrail 11: DELETE_FAILED Usually Means a Resource Cannot Be Deleted
Common causes: S3 bucket is not empty, ENI is in use by another resource, security group is referenced by another security group or ENI, Lambda function is associated with a CloudFront distribution. Fix the dependency first, then retry the delete. Use `--retain-resources` to skip undeletable resources.

## Guardrail 12: CDK Synth vs Deploy — Local vs Remote Errors
CDK synth errors are local template generation failures — the CDK app code has bugs, missing context, or construct library issues. CDK deploy errors are remote CloudFormation failures — the generated template is valid but the AWS environment rejects it. Always check which phase failed before troubleshooting.
