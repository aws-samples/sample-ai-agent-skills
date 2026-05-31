---
title: "D2 — Query Log Analysis"
description: "Diagnose DNS issues using Route 53 Resolver query log data"
status: active
severity: MEDIUM
triggers: ["analyze DNS queries", "DNS query patterns", "query log investigation"]
owner: devops-agent
objective: "Use query logs to diagnose DNS resolution issues and identify patterns"
context: "Query logs contain DNS query details including source IP, domain, query type, response code, and resolver endpoint used. Logs are in JSON format and can be analyzed via CloudWatch Logs Insights, Athena, or direct S3 analysis."
---

## Phase 1 — Triage

MUST:
- Verify query logging is active: `aws route53resolver list-resolver-query-log-config-associations --filters Name=ResourceId,Values=<vpc-id>`
- Check log destination for recent entries
- For CloudWatch Logs: `aws logs filter-log-events --log-group-name <group> --filter-pattern '<domain>' --limit 20`

SHOULD:
- Identify the time range of the DNS issue
- Check for SERVFAIL, NXDOMAIN, or timeout response codes in logs

MAY:
- Use CloudWatch Logs Insights for complex query analysis
- Export logs to S3 for Athena-based analysis

## Phase 2 — Remediate

MUST:
- Identify the failing DNS queries from logs (domain, response code, source IP)
- Correlate query failures with resolver endpoint or forwarding rule issues
- Route to the appropriate specific runbook based on log analysis findings

SHOULD:
- Check if SERVFAIL responses correlate with DNSSEC validation issues
- Identify if specific source IPs or instances are generating problematic queries

MAY:
- Set up CloudWatch metric filters for DNS failure patterns
- Create dashboards for ongoing DNS health monitoring

## Common Issues

- symptoms: "Query logs show SERVFAIL for specific domains"
  diagnosis: "DNSSEC validation failure or target DNS server not responding."
  resolution: "Check DNSSEC validation status. Verify forwarding rule target servers are healthy."

- symptoms: "Query logs show high volume of NXDOMAIN responses"
  diagnosis: "Applications querying non-existent domains or misconfigured DNS search domains."
  resolution: "Review application DNS configuration. Check DHCP option set search domains."

## Safety Ratings

```
safety_ratings:
  - "list-resolver-query-log-config-associations: GREEN — read-only check"
  - "logs filter-log-events: GREEN — read-only log query"
  - "logs start-query (Insights): GREEN — read-only log analysis"
  - "athena start-query-execution: GREEN — read-only data analysis"
```

## Escalation Conditions

- Query logs reveal DNS exfiltration patterns (high-entropy subdomain queries)
- Massive query volume from specific instances suggests DNS amplification
- Query failures affect critical production services
- Log analysis reveals unauthorized DNS resolution attempts

## Data Sensitivity

- DNS query logs: HIGH — contain all DNS queries, source IPs, and response data
- CloudWatch Logs Insights results: HIGH — aggregated DNS query data
- Athena query results: HIGH — may contain sensitive DNS patterns
- Source IP correlation: MEDIUM — can identify specific instances and their DNS behavior

## Prohibited Actions

- NEVER share raw query logs without redacting sensitive domain names and IP addresses
- NEVER delete query logs that may be needed for security investigation
- NEVER ignore DNS exfiltration patterns found in query logs
- NEVER assume query log absence means no DNS issues — logging may not be enabled

## Phase 3 — Rollback

1. No rollback needed for read-only log analysis
2. If metric filters or dashboards were created incorrectly, delete them
3. If Athena tables were created, clean up if no longer needed
4. Preserve all query logs related to security investigations
5. Document analysis findings and recommendations

## Output Format

```yaml
root_cause: "query_log_analysis — <specific_cause>"
evidence:
  - type: log_analysis
    content: "<query log findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified DNS issue"
  long_term: "Implement continuous DNS monitoring via query logs"
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
