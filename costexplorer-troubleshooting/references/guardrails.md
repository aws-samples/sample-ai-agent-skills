# Cost Explorer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Cost Explorer or Budgets issue.

## Guardrail 1: Cost Data Has 24-48 Hour Delay
Cost Explorer data is not real-time. There is a 24-48 hour delay before costs appear. Do not diagnose missing recent costs as an error. Always check the data availability window before concluding costs are missing.

## Guardrail 2: Cost Allocation Tags Must Be Activated
Creating tags on AWS resources does not automatically make them cost allocation tags. Tags must be explicitly activated in the Billing and Cost Management console. Activation takes up to 24 hours to take effect. Only activated tags appear in Cost Explorer and CUR.

## Guardrail 3: Budgets Evaluate Periodically Not Real-Time
AWS Budgets evaluate cost data periodically (typically once per day for cost budgets). Budget alerts are not instant. There can be a delay of up to 24 hours between exceeding a threshold and receiving the alert. Never claim budget alerts are real-time.

## Guardrail 4: Anomaly Detection Needs Historical Baseline
Cost Anomaly Detection uses machine learning that requires at least 2 weeks of historical spending data to establish a baseline. New accounts or newly monitored services will not generate meaningful anomaly alerts immediately. Never expect instant anomaly detection on new setups.

## Guardrail 5: Unblended vs Blended vs Amortized Costs
Unblended cost is the actual charge. Blended cost averages RI/SP discounts across the organization. Amortized cost spreads upfront payments across the commitment term. Using the wrong metric leads to incorrect cost analysis. Always clarify which metric is being used.

## Guardrail 6: CUR Reports Have Delivery Delay
Cost and Usage Reports are delivered to S3 with up to 24-hour delay. The first report after creation may take up to 24 hours to appear. Reports are updated multiple times per day as cost data finalizes. Never diagnose a missing first report within 24 hours as an error.

## Guardrail 7: Linked Account Access Must Be Enabled
In AWS Organizations, member accounts cannot access Cost Explorer by default. The management account must enable Cost Explorer access for member accounts. Without this, linked accounts see no cost data. This is a configuration requirement, not a bug.

## Guardrail 8: Savings Plans Apply Automatically
Savings Plans discounts are applied automatically to eligible usage. You cannot control which specific resources receive the discount. The discount applies to the highest-savings usage first. Never suggest manually assigning Savings Plans to specific instances.

## Guardrail 9: Reserved Instance Utilization Includes Unused Hours
RI utilization measures how much of the reserved capacity is being used. Unused RI hours still incur cost. Low utilization means you are paying for capacity you are not using. Never confuse RI utilization with cost savings.

## Guardrail 10: Budget Actions Require IAM Roles
Budget actions (applying SCPs, IAM policies, or targeting instances) require an IAM role with appropriate permissions. The budget service must be able to assume this role. Missing or misconfigured roles cause silent action failures.

## Guardrail 11: Cost Categories Take 24 Hours to Apply
Cost categories are custom groupings for cost organization. After creation or modification, they take up to 24 hours to apply to cost data. Historical data is not retroactively categorized. Never expect immediate cost category application.

## Guardrail 12: Forecast Accuracy Depends on Spending Patterns
Cost Explorer forecasts use historical spending patterns. Irregular or recently changed spending patterns reduce forecast accuracy. New services or significant usage changes make forecasts unreliable for 2-4 weeks. Never treat forecasts as guarantees.
