---
title: "D1 — Query Logging Configuration"
description: "Diagnose Route 53 Resolver query logging setup and delivery issues"
status: active
severity: MEDIUM
triggers: ["query logging not working", "DNS logs missing", "query log configuration failed"]
owner: devops-agent
objective: "Resolve query logging configuration and log delivery issues"
context: "Query logging captures all DNS queries in associated VPCs. Logs can be sent to CloudWatch Logs, S3, or Kinesis Data Firehose. Logging captures all VPC DNS queries, not just resolver endpoint queries."
---

## Phase 1 — Triage

MUST:
- List query log configs: `aws route53resolver list-resolver-query-log-configs --query 'ResolverQueryLogConfigs[*].{Id:Id,Name:Name,Status:Status,AssociationCount:AssociationCount,DestinationArn:DestinationArn}'`
- Check config associations: `aws route53resolver list-resolver-query-log-config-associations --query 'ResolverQueryLogConfigAssociations[*].{ConfigId:ResolverQueryLogConfigId,ResourceId:ResourceId,Status:Status,Error:Error}'`
- Verify destination resource exists and is accessible

SHOULD:
- Check IAM permissions for the log destination (CloudWatch Logs, S3, or Firehose)
- Verify the query log config status is CREATED (not CREATING or FAILED)

MAY:
- Check CloudWatch Logs for recent log entries if destination is CloudWatch
- Review CloudTrail for CreateResolverQueryLogConfig errors

## Phase 2 — Remediate

MUST:
- Create query log config if missing: `aws route53resolver create-resolver-query-log-config --name <name> --destination-arn <arn>`
- Associate with VPC: `aws route53resolver associate-resolver-query-log-config --resolver-query-log-config-id <config-id> --resource-id <vpc-id>`
- Ensure the destination resource (log group, S3 bucket, Firehose stream) exists and has correct permissions

SHOULD:
- Configure log retention policy for CloudWatch Logs destinations
- Set up S3 lifecycle rules for S3 destinations to manage costs

MAY:
- Create query log configs in multiple regions for multi-region VPCs
- Set up CloudWatch alarms on query log metrics

## Common Issues

- symptoms: "Query log config created but no logs appearing"
  diagnosis: "Config not associated with any VPC or destination permissions incorrect."
  resolution: "Associate config with VPC. Verify IAM resource policy on destination allows Route 53 Resolver."

- symptoms: "Query log association status shows FAILED"
  diagnosis: "Destination resource does not exist or IAM permissions are insufficient."
  resolution: "Verify destination ARN is correct. Check resource policy allows logs:CreateLogStream and logs:PutLogEvents."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-query-log-configs: GREEN — read-only listing"
  - "create-resolver-query-log-config: YELLOW — creates logging resource with cost implications"
  - "associate-resolver-query-log-config: YELLOW — enables DNS query logging for VPC"
  - "delete-resolver-query-log-config: YELLOW — stops DNS query logging"
```

## Escalation Conditions

- Query logging causes performance degradation in the VPC
- Log volume exceeds storage capacity or budget
- Compliance requires query logging but configuration fails
- Cross-account log delivery fails despite correct RAM sharing

## Data Sensitivity

- `aws route53resolver list-resolver-query-log-configs`: LOW — config metadata only
- DNS query logs: HIGH — contain all DNS queries including internal service names and IP addresses
- Log destination ARNs: MEDIUM — reveal logging infrastructure
- Query log associations: LOW — VPC and config ID mappings

## Prohibited Actions

- NEVER enable query logging in production without estimating log volume and costs
- NEVER store DNS query logs in unencrypted S3 buckets
- NEVER delete query log configs that are required for compliance
- NEVER share query logs with unauthorized parties — they contain sensitive DNS data

## Phase 3 — Rollback

1. If logging was enabled incorrectly, disassociate: `aws route53resolver disassociate-resolver-query-log-config --resolver-query-log-config-id <config-id> --resource-id <vpc-id>`
2. If the config was created incorrectly, delete it: `aws route53resolver delete-resolver-query-log-config --resolver-query-log-config-id <config-id>`
3. If log destination was created, evaluate retention and cleanup
4. Verify no compliance requirements are violated by disabling logging
5. Document the logging changes and notify the security team

## Output Format

```yaml
root_cause: "query_logging_configuration — <specific_cause>"
evidence:
  - type: logging_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix query logging configuration"
  long_term: "Automate logging setup via IaC with proper retention policies"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"
