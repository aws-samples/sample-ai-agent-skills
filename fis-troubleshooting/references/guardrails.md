# AWS Fault Injection Simulator Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any FIS issue.

## Guardrail 1: FIS Causes Real Failures
FIS experiments cause ACTUAL failures in your infrastructure — instances are stopped, network is disrupted, latency is injected. These are not simulations. Always have stop conditions and rollback plans before running experiments.

## Guardrail 2: IAM Role Needs Both FIS and Resource Permissions
The FIS experiment role needs fis:* permissions AND permissions to perform actions on target resources (ec2:StopInstances, ecs:UpdateService, etc.). Missing resource permissions cause action failures even if the experiment starts successfully.

## Guardrail 3: Stop Conditions Require Pre-Existing CloudWatch Alarms
Stop conditions reference CloudWatch alarm ARNs. The alarms must exist before the experiment starts. FIS does not create alarms. If the referenced alarm doesn't exist, the experiment may run without safety guardrails.

## Guardrail 4: Target Filters Must Match Existing Resources
Target selection uses filters (tags, resource IDs, ARNs). If filters match zero resources, the experiment fails with target resolution error. Verify filters match at least one resource before starting experiments.

## Guardrail 5: Some Actions Are Irreversible
Actions like aws:ec2:terminate-instances permanently terminate instances. FIS does NOT restore terminated resources after experiment completion. Use stop-instances instead of terminate for recoverable experiments. Always understand action reversibility.

## Guardrail 6: Actions Execute in Parallel by Default
Without explicit dependencies (startAfter), all actions in an experiment execute simultaneously. This can cause cascading failures. Use action dependencies to control execution order when needed.

## Guardrail 7: Experiment Duration Limits Action Scope
The experiment has a maximum duration. If actions take longer than the experiment duration, they may be interrupted. Set experiment duration longer than the longest action plus recovery time.

## Guardrail 8: Selection Mode Affects Blast Radius
Target selection mode (ALL, COUNT(n), PERCENT(p)) determines how many matching resources are affected. ALL affects every matching resource. Use COUNT or PERCENT to limit blast radius during initial testing.

## Guardrail 9: Multi-Account Experiments Need Cross-Account Setup
Running experiments across accounts requires target account configurations in the template and cross-account IAM roles. The FIS role in the orchestrator account must be able to assume roles in target accounts.

## Guardrail 10: Experiment Logs Are Essential for Post-Mortem
FIS experiment results include action states and timestamps. Always review experiment results after completion. Failed actions may indicate infrastructure issues beyond the intended fault injection.

## Guardrail 11: Network Actions Affect All Traffic in Scope
Network disruption actions (aws:ec2:disrupt-network) affect ALL traffic matching the scope, not just test traffic. This can impact production users if the target scope is too broad.

## Guardrail 12: FIS Has Service Quotas
FIS has limits on concurrent experiments, experiment templates, and actions per template. Check service quotas if you hit limits: aws service-quotas list-service-quotas --service-code fis.
