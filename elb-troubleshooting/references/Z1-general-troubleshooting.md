---
title: "Z1 — General ELB Troubleshooting"
description: "General troubleshooting guide for ELB issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "ELB error"
  - "load balancer issue"
  - "ALB troubleshooting"
  - "NLB troubleshooting"
  - "CLB troubleshooting"
owner: devops-agent
objective: "Provide a general troubleshooting framework for ELB issues and route to specific runbooks"
context: "This runbook covers general ELB troubleshooting when the issue does not clearly map to a specific runbook. It provides a systematic diagnostic framework covering ALB, NLB, and CLB. Use this as a starting point and pivot to specific runbooks once the issue category is identified. ELB types: ALB (Layer 7 HTTP/HTTPS), NLB (Layer 4 TCP/UDP/TLS), CLB (Legacy Layer 4/7)."
---

## Phase 1 — Triage

MUST:
- Identify the load balancer type and configuration: `aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn>`
- Check load balancer state: `aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn> --query 'LoadBalancers[0].State'`
- Check all target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>` (for each target group)
- Review CloudWatch error metrics: HTTPCode_ELB_4XX_Count, HTTPCode_ELB_5XX_Count (ALB), TCP_ELB_Reset_Count (NLB)
- Check listener configuration: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`

SHOULD:
- Verify security groups (ALB only): `aws ec2 describe-security-groups --group-ids <lb-sg-id>`
- Check if access logging is enabled for detailed analysis
- Review recent changes via CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticloadbalancing.amazonaws.com --max-results 20`
- Verify DNS resolution of the LB endpoint: `nslookup <lb-dns-name>`

MAY:
- Check AWS Health Dashboard for ELB service issues
- Review the LB's AZ configuration and subnet assignments
- Check if WAF is associated (ALB only) and potentially blocking traffic

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings:
  - Target health issues → A1 (Target Health) or A2 (Registration)
  - HTTP errors → B1 (502), B2 (503), or B3 (504)
  - TLS/certificate issues → C1 (SSL/TLS) or C2 (Certificates)
  - Session/routing issues → D1 (Sticky Sessions) or D2 (Cross-Zone)
  - Logging/WAF issues → E1 (Access Logs) or E2 (WAF)
  - NLB/draining issues → F1 (NLB TCP) or F2 (Connection Draining)
- If the issue spans multiple categories, address the most impactful component first

SHOULD:
- Document the troubleshooting steps and findings
- Verify the fix resolves the root cause, not just the symptom
- Check for related issues across target groups and listeners

MAY:
- Create a post-incident report for significant outages
- Update monitoring and alerting based on the issue discovered
- Propose architectural improvements to prevent recurrence

## Common Issues

- symptoms: "Load balancer DNS name not resolving"
  diagnosis: "LB is in provisioning state, or DNS propagation delay."
  resolution: "Wait for LB to become active. Verify the DNS name is correct."

- symptoms: "Intermittent connectivity issues"
  diagnosis: "AZ-specific problems, target flapping, or client-side DNS caching."
  resolution: "Check per-AZ health. Verify target stability. Use short DNS TTL."

- symptoms: "High latency through the load balancer"
  diagnosis: "Target processing time, cross-AZ traffic, or LB capacity."
  resolution: "Check TargetResponseTime metric. Enable cross-zone. Scale targets."

- symptoms: "CLB needs migration to ALB/NLB"
  diagnosis: "CLB is legacy and lacks features of ALB/NLB."
  resolution: "Use the AWS ELB migration tool or manually recreate with ALB/NLB."

## Output Format

```yaml
root_cause: "ELB issue — <specific_cause>"
evidence:
  - type: lb_status
    content: "<load balancer state and configuration>"
  - type: error_metrics
    content: "<CloudWatch error count metrics>"
severity: MEDIUM
mitigation:
  immediate: "<specific remediation based on root cause>"
  long_term: "Implement comprehensive ELB monitoring and alerting"
routed_to: "<specific runbook ID if applicable>"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 remediation varies by specific runbook: YELLOW to RED"
  - "Refer to the specific runbook's safety ratings for detailed guidance"

## Escalation Conditions

escalation_conditions:
  - "Issue cannot be categorized into a specific runbook after triage"
  - "Multiple LB components are failing simultaneously"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves potential security breach or DDoS attack"
  - "AWS service-side issue suspected"

## Data Sensitivity

data_sensitivity:
  - command: "describe-load-balancers"
    sensitivity: MEDIUM
    contains: "LB configuration, DNS names, and VPC details"
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API call details including principal ARNs"
  - command: "access logs"
    sensitivity: HIGH
    contains: "Client IPs, request paths, and response details"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete a production load balancer without explicit confirmation"
  - "NEVER modify security groups to allow unrestricted access"
  - "NEVER disable WAF on production ALBs without security approval"
  - "NEVER change listener protocols without understanding the impact"

## Phase 3 — Rollback

- Document the pre-change state before any remediation
- Follow the rollback procedures in the specific runbook used for remediation
- If multiple changes were made, roll back in reverse order
- Verify LB health, target health, and traffic flow after rollback
- Retain logs and metrics for post-incident analysis
