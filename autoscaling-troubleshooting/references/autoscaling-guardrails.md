# EC2 Auto Scaling Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EC2 Auto Scaling issue.

## Guardrail 1: Health Check Grace Period Is NOT Cooldown
The health check grace period delays health evaluation for newly launched instances. The cooldown period prevents additional scaling actions after a scaling activity. These are completely different mechanisms. Grace period protects new instances from premature termination. Cooldown prevents scaling oscillation. Do not confuse or conflate them.

## Guardrail 2: ELB Health Checks Must Be Explicitly Enabled
By default, Auto Scaling groups use EC2 status checks only. ELB health checks must be explicitly enabled by setting HealthCheckType to ELB. When enabled, an instance must pass BOTH EC2 and ELB health checks. An instance passing EC2 checks but failing ELB checks will be terminated. Always verify HealthCheckType in the ASG configuration.

## Guardrail 3: Scaling Activities Can Fail Without Visible Errors
Scaling activity failures are recorded in describe-scaling-activities but may not trigger alarms by default. Common failure causes include: insufficient IAM permissions, no capacity in the AZ, instance type unavailable, launch template referencing deleted resources, security group or subnet not found. Always check StatusCode and StatusMessage.

## Guardrail 4: Cooldown Behavior Differs by Policy Type
Default cooldown applies to simple scaling policies. Target tracking policies have their own scale-in and scale-out cooldowns that override the default. Step scaling policies can disable cooldown entirely. Scheduled actions ignore cooldown. Understanding which cooldown applies is critical for diagnosing scaling delays.

## Guardrail 5: Multiple Sources Can Modify Desired Capacity
Scaling policies, scheduled actions, manual updates via console/CLI, CloudFormation, Terraform, and external automation can all modify desired capacity. Conflicting modifications cause oscillation. Check CloudTrail for SetDesiredCapacity and UpdateAutoScalingGroup calls to identify competing sources.

## Guardrail 6: Launch Templates and Launch Configurations Are Different
Launch templates support versioning, multiple instance types, and mixed instances policies. Launch configurations are legacy and immutable — any change requires creating a new one. You cannot use mixed instances policies with launch configurations. Always prefer launch templates for new ASGs.

## Guardrail 7: Mixed Instances Allocation Strategies Affect Instance Selection
On-Demand allocation uses "prioritized" (default) or "lowest-price" strategy. Spot allocation uses "capacity-optimized" (recommended), "capacity-optimized-prioritized", "lowest-price", or "price-capacity-optimized". The strategy determines which instance types are launched. Insufficient instance type diversity causes capacity failures.

## Guardrail 8: Warm Pool Instances Have Their Own Lifecycle
Warm pool instances transition through: Warmed:Pending → Warmed:Pending:Wait (if lifecycle hook) → Warmed:Pending:Proceed → Warmed:Running/Stopped/Hibernated. When needed, they transition to InService. Warm pool has its own MaxGroupPreparedCapacity. Lifecycle hooks can apply to warm pool transitions separately.

## Guardrail 9: Instance Refresh Respects MinHealthyPercentage
Instance refresh replaces instances in batches. MinHealthyPercentage (default 90%) determines how many instances must remain healthy during refresh. Setting it too high slows the refresh. Setting it too low risks availability. A failed health check during refresh can pause the entire operation. SkipMatching can avoid unnecessary replacements.

## Guardrail 10: Predictive Scaling Requires Sufficient Historical Data
Predictive scaling needs at least 24 hours of historical data but works best with 14 days. Forecasts are generated daily and look 48 hours ahead. It works best with recurring, predictable patterns. It should be combined with dynamic scaling for unexpected spikes. ForecastOnly mode lets you evaluate before enabling scaling.

## Guardrail 11: Lifecycle Hook Timeout Causes Automatic Action
When a lifecycle hook times out (default 3600 seconds, max 7200), the default action (CONTINUE or ABANDON) executes automatically. ABANDON on launch cancels the instance. CONTINUE on launch proceeds to InService. Heartbeat timeout can be extended with record-lifecycle-action-heartbeat. Always monitor hook completion.

## Guardrail 12: Termination Policy Evaluation Order Matters
Auto Scaling first identifies the AZ with the most instances (for rebalancing). Then it applies the termination policy within that AZ. Multiple policies are evaluated in order. Default policy: OldestLaunchTemplate → OldestLaunchConfiguration → ClosestToNextInstanceHour. Custom policies can use Lambda functions. The order significantly affects which instances are terminated.
