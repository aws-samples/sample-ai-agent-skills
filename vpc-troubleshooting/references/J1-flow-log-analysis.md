---
title: "J1 — Flow Log Analysis"
description: "Guide for analyzing VPC flow logs to diagnose connectivity issues"
status: active
severity: MEDIUM
triggers:
  - "Need to analyze flow logs"
  - "Connectivity debugging"
  - "Traffic analysis"
owner: devops-agent
objective: "Use flow logs to identify traffic patterns and connectivity issues"
context: "VPC flow logs capture IP traffic at ENI level. They show ACCEPT/REJECT, source/dest IP+port, protocol, packets, bytes. They do NOT show packet content, DNS queries (use Route 53 query logs), or which specific rule matched."
---

## Phase 1 — Triage

MUST:
- Verify flow logs are enabled: `aws ec2 describe-flow-logs --filter Name=resource-id,Values=<vpc-id or subnet-id or eni-id>`
- Query flow logs for the relevant time period and IPs
- Filter for REJECT entries to find blocked traffic
- Check both source and destination ENI flow logs

SHOULD:
- Correlate REJECT entries with SG/NACL rules
- Check for asymmetric patterns (traffic in one direction only)
- Look for unexpected source IPs or ports

MAY:
- Use CloudWatch Logs Insights for complex queries
- Export to S3 and use Athena for large-scale analysis
- Check flow log format version (v2-v5 have different fields)

## Common Issues

- symptoms: "Flow logs show REJECT but SG allows the traffic"
  diagnosis: "NACL is blocking the traffic. Flow logs show the final verdict, not which layer blocked."
  resolution: "Check NACL rules for the subnet. Remember NACLs are stateless."

- symptoms: "No flow log entries for expected traffic"
  diagnosis: "Flow logs not enabled on the correct resource, or traffic never reached the ENI."
  resolution: "Enable flow logs on the specific ENI, check routing to ensure traffic reaches the ENI."

## Safety Ratings

```
safety_ratings:
  - "describe-flow-logs: GREEN — read-only flow log configuration check"
  - "filter-log-events (CloudWatch Logs): GREEN — read-only flow log query"
  - "start-query (CloudWatch Logs Insights): GREEN — read-only log analysis"
  - "create-flow-logs (enable flow logs): YELLOW — enables logging, recoverable by deleting"
  - "delete-flow-logs: YELLOW — disables logging, recoverable by re-creating"
```

## Escalation Conditions

- "Flow log analysis reveals unauthorized access attempts"
- "Flow logs show REJECT patterns affecting production services"
- "Fix requires enabling flow logs on production VPC (cost implications)"
- "Analysis reveals unexpected traffic patterns requiring security review"

## Data Sensitivity

- HIGH: VPC flow logs (contain source/destination IPs, ports, and traffic patterns)
- HIGH: flow log analysis results (expose network architecture and access patterns)
- MEDIUM: flow log configuration, CloudWatch log group settings

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER share raw flow log data containing source/destination IPs without authorization"

## Phase 3 — Rollback

- If flow logs were enabled: delete them with `aws ec2 delete-flow-logs --flow-log-ids <fl-id>` (historical data remains in CloudWatch/S3)
- If flow logs were deleted: re-create them with `aws ec2 create-flow-logs --resource-ids <resource-id> --resource-type <VPC|Subnet|NetworkInterface> --traffic-type ALL --log-destination-type <cloud-watch-logs|s3>`
- No rollback needed for read-only analysis operations
- Document flow log configuration and log group/S3 destination before changes

## Output Format

```yaml
root_cause: "flow_log_finding — <detail>"
evidence:
  - type: flow_log
    content: "<relevant flow log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix the identified blocking rule"
  long_term: "Enable flow logs on all VPCs, set up automated analysis"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
