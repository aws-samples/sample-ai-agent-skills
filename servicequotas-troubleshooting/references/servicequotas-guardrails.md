# AWS Service Quotas Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Service Quotas issue.

## Guardrail 1: Not All Quotas Are Adjustable
Some quotas are hard limits that cannot be increased through Service Quotas or support tickets. Check the Adjustable field in get-service-quota response. Non-adjustable quotas require architectural changes (e.g., distributing across regions or accounts). Do not promise quota increases for non-adjustable limits.

## Guardrail 2: Applied Quota May Differ from Default
If a quota increase was previously granted, the applied value (get-service-quota) differs from the default (get-aws-default-service-quota). Always check the applied value for the current effective limit. The default value shows what new accounts get. Previous increases are account-specific.

## Guardrail 3: Quota Increase Requests Can Take Days
Some quota increases are approved automatically within minutes (within pre-approved limits). Others require manual review by AWS service teams and can take 1-5 business days. Critical production needs should be planned ahead. Check request status with list-requested-service-quota-change-history.

## Guardrail 4: CloudWatch Usage Metrics Are Not Available for All Quotas
Only some services publish usage metrics to CloudWatch under the AWS/Usage namespace. Check the UsageMetric field in the quota details. If no usage metric exists, you must calculate utilization manually using service-specific APIs (e.g., describe-instances for EC2).

## Guardrail 5: Organization Quota Templates Apply to New Accounts Only
Quota templates set default quotas for new accounts joining the organization. They do NOT retroactively change quotas in existing accounts. To change quotas in existing accounts, submit individual increase requests per account. Templates require Organizations integration.

## Guardrail 6: Some Quotas Are Regional, Others Are Global
EC2 instance limits, VPC limits, and most service quotas are per-region. IAM limits (users, roles, policies) are global. Always check the correct region when looking up or requesting quota changes. A quota increase in us-east-1 does not affect us-west-2.

## Guardrail 7: Service Quotas API Does Not Cover All Services
Some older or specialized services manage quotas through their own APIs or require support tickets. Check if the service appears in list-services. If not, use the service's own API or contact AWS Support for quota management.

## Guardrail 8: Quota Utilization Can Spike Temporarily
Some services allow brief bursts above quota limits. Sustained usage above quota will be throttled or denied. A momentary spike in utilization does not necessarily mean you need a quota increase. Monitor sustained utilization patterns before requesting increases.

## Guardrail 9: EC2 On-Demand Limits Are vCPU-Based Per Instance Family
EC2 On-Demand instance limits are measured in vCPUs, not instance count. Limits are per instance family: Standard (A, C, D, H, I, M, R, T, Z), Accelerated (F, G, Inf, P, VT), etc. Running 100 t3.micro (2 vCPUs each) uses 200 vCPUs from the Standard family quota.

## Guardrail 10: Lambda Concurrency Is Account-Level Per Region
Lambda concurrent execution quota is shared across all functions in an account per region. Default is 1,000. Reserved concurrency for one function reduces available concurrency for others. Provisioned concurrency counts against the total. Monitor with CloudWatch ConcurrentExecutions metric.

## Guardrail 11: VPC Resource Limits Are Per-VPC and Per-Region
VPC quotas include subnets per VPC, security groups per VPC, rules per security group, route tables per VPC, and VPCs per region. Some are per-VPC, others per-region. Exceeding any limit prevents resource creation. Check specific quota codes for each resource type.

## Guardrail 12: Quota Increase Approval Depends on Multiple Factors
AWS considers account history, current usage patterns, the specific service, and the requested increase amount. Very large increases may require justification. New accounts may have lower initial limits. Gradual increases over time are more likely to be approved than sudden large requests.
