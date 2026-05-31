---
title: "F1 — Load Balancer Configuration Issues"
description: "Diagnose Lightsail load balancer setup and health check problems"
status: active
severity: HIGH
triggers:
  - "load balancer error"
  - "health check failing"
  - "instances not healthy"
  - "LB not routing traffic"
owner: devops-agent
objective: "Identify and resolve Lightsail load balancer configuration issues"
context: "Lightsail load balancers distribute HTTPS traffic across instances. They include a free SSL/TLS certificate. Health checks determine instance availability. LBs support up to 5 target instances. Health check path must return HTTP 200."
---

## Phase 1 — Triage

MUST:
- Check LB status: `aws lightsail get-load-balancers --query 'loadBalancers[*].{Name:name,State:state,DNSName:dnsName,Instances:instancePort}'`
- Check LB details: `aws lightsail get-load-balancer --load-balancer-name <lb-name>`
- Check instance health: `aws lightsail get-load-balancer-metric-data --load-balancer-name <lb-name> --metric-name HealthyHostCount --period 300 --start-time <start> --end-time <end> --unit Count --statistics Average`
- Verify health check path configuration

SHOULD:
- Check attached instances and their states
- Verify health check path returns HTTP 200
- Check for 5XX errors: HTTPCode_Target_5XX_Count metric

MAY:
- Review LB access logs
- Check session stickiness configuration

## Phase 2 — Remediate

MUST:
- Ensure health check path returns HTTP 200 from all instances
- Attach instances: `aws lightsail attach-instances-to-load-balancer --load-balancer-name <lb-name> --instance-names <instance-names>`
- Verify instances are in running state

SHOULD:
- Configure appropriate health check path: `aws lightsail update-load-balancer-attribute --load-balancer-name <lb-name> --attribute-name HealthCheckPath --attribute-value /health`
- Point DNS to load balancer DNS name

MAY:
- Enable session stickiness if needed
- Configure custom HTTP to HTTPS redirect

## Common Issues

- symptoms: "All instances showing unhealthy"
  diagnosis: "Health check path not returning HTTP 200."
  resolution: "Verify health check path exists and returns 200. Check application is running."

- symptoms: "Load balancer not accessible"
  diagnosis: "DNS not pointing to LB or LB not active."
  resolution: "Point domain CNAME to LB DNS name. Verify LB is active."

- symptoms: "Intermittent 502/503 errors"
  diagnosis: "Some instances failing health checks or application errors."
  resolution: "Check instance health. Review application logs for errors."

## Output Format

```yaml
root_cause: "lb_config — <specific_cause>"
evidence:
  - type: lb_status
    content: "<load balancer state>"
  - type: instance_health
    content: "<healthy/unhealthy instance count>"
  - type: health_check
    content: "<health check path and results>"
severity: HIGH
mitigation:
  immediate: "Fix health check configuration and instance health"
  long_term: "Implement proper health check endpoints and monitoring"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
