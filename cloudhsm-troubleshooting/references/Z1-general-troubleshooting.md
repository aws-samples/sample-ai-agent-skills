---
title: "Z1 — General CloudHSM Troubleshooting"
description: "General troubleshooting guide for AWS CloudHSM issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "CloudHSM error"
  - "HSM issue"
  - "CloudHSM troubleshooting"
  - "CloudHSM general"
owner: devops-agent
objective: "Provide a general troubleshooting framework for CloudHSM issues and route to specific runbooks"
context: "This runbook covers general CloudHSM troubleshooting approaches when the issue does not clearly map to a specific runbook. It provides a systematic diagnostic framework covering cluster health, client connectivity, user management, key operations, and integration issues. Use this as a starting point and pivot to specific runbooks once the issue category is identified."
---

## Phase 1 — Triage

MUST:
- Check overall cluster health: `aws cloudhsmv2 describe-clusters --query 'Clusters[*].{ID:ClusterId,State:State,HSMs:Hsms[*].{ID:HsmId,State:State,AZ:AvailabilityZone}}'`
- Verify CloudHSM client daemon status: `sudo systemctl status cloudhsm-client`
- Check client logs for errors: `sudo journalctl -u cloudhsm-client --no-pager -n 100`
- Verify network connectivity to HSM ENIs: `nc -zv <hsm-eni-ip> 2223`
- Check CloudTrail for recent CloudHSM API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudhsmv2.amazonaws.com --max-results 20`

SHOULD:
- Verify IAM permissions for CloudHSM API calls
- Check AWS Health Dashboard for CloudHSM service issues
- Review recent changes to VPC, security groups, or NACLs
- Verify the CloudHSM SDK version is current

MAY:
- Check CloudWatch metrics for the cluster (if custom metrics are configured)
- Review application-level logs for HSM-related errors
- Check if the issue correlates with recent deployments or infrastructure changes

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings:
  - Cluster/HSM issues → A1 (Cluster Creation) or A2 (HSM Provisioning)
  - Connectivity issues → B1 (Client Connectivity) or B2 (CLI Errors)
  - Key issues → C1 (Key Management) or C2 (Key Import/Export)
  - User issues → D1 (User Management) or D2 (Quorum Authentication)
  - Integration issues → E1 (SSL Offload) or E2 (JCE Provider)
  - Availability issues → F1 (High Availability) or F2 (Cross-Region Backup)
  - SDK issues → G1 (SDK Issues)
- If the issue spans multiple categories, address the most critical component first

SHOULD:
- Document the troubleshooting steps taken and findings
- Verify the fix resolves the root cause, not just the symptom
- Check for related issues that may have the same root cause

MAY:
- Create a post-incident report for significant outages
- Update monitoring and alerting based on the issue discovered
- Propose preventive measures to avoid recurrence

## Common Issues

- symptoms: "General HSM operations are slow"
  diagnosis: "Single HSM under heavy load, network latency, or client SDK performance issue."
  resolution: "Add more HSMs for throughput. Check network path. Update SDK to latest version."

- symptoms: "CloudHSM API calls return throttling errors"
  diagnosis: "API rate limits exceeded for CloudHSM management operations."
  resolution: "Implement exponential backoff and retry. Reduce API call frequency."

- symptoms: "Cluster shows DEGRADED state"
  diagnosis: "One or more HSMs have issues but the cluster is still operational."
  resolution: "Check individual HSM states. Replace degraded HSMs. See A2 runbook."

- symptoms: "Applications fail after AWS maintenance window"
  diagnosis: "HSM firmware update or maintenance may have changed behavior."
  resolution: "Check AWS notifications for maintenance details. Verify client SDK compatibility."

## Output Format

```yaml
root_cause: "CloudHSM issue — <specific_cause>"
evidence:
  - type: cluster_health
    content: "<describe-clusters output>"
  - type: client_status
    content: "<client daemon status and logs>"
severity: MEDIUM
mitigation:
  immediate: "<specific remediation based on root cause>"
  long_term: "Implement comprehensive CloudHSM monitoring and alerting"
routed_to: "<specific runbook ID if applicable>"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list/logs): GREEN — read-only"
  - "Phase 2 remediation varies by specific runbook: YELLOW to RED"
  - "Refer to the specific runbook's safety ratings for detailed guidance"

## Escalation Conditions

escalation_conditions:
  - "Issue cannot be categorized into a specific runbook after triage"
  - "Multiple subsystems are failing simultaneously"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves potential security breach or key compromise"
  - "AWS service-side issue suspected (internal errors, degraded service)"

## Data Sensitivity

data_sensitivity:
  - command: "describe-clusters"
    sensitivity: HIGH
    contains: "Cluster topology, HSM IDs, and ENI details"
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API call details including principal ARNs and parameters"
  - command: "client daemon logs"
    sensitivity: MEDIUM
    contains: "Connection details and operational errors"

## Prohibited Actions

prohibited_actions:
  - "NEVER attempt to access HSM hardware directly"
  - "NEVER share cluster or HSM identifiers with unauthorized parties"
  - "NEVER modify HSM ENIs through EC2 APIs"
  - "NEVER disable CloudHSM automatic backups"
  - "NEVER bypass quorum authentication requirements"

## Phase 3 — Rollback

- Document the pre-change state before any remediation
- Follow the rollback procedures in the specific runbook used for remediation
- If multiple changes were made, roll back in reverse order
- Verify cluster health and application functionality after rollback
- Retain logs and evidence for post-incident analysis
