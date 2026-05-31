---
title: "E2 — License Endpoint Errors"
description: "Diagnose license endpoint configuration and connectivity issues"
status: active
severity: HIGH
triggers: ["license error", "license endpoint failed", "cannot reach license server", "license checkout failed"]
owner: devops-agent
objective: "Resolve license endpoint issues for commercial DCC application licensing"
context: "License endpoints connect Deadline Cloud workers to license servers for commercial applications (Maya, Houdini, Nuke, etc.). Workers must have network access to the license server through the endpoint."
---

## Phase 1 — Triage

MUST:
- List license endpoints: `aws deadline list-license-endpoints --query 'licenseEndpoints[*].{LicenseEndpointId:licenseEndpointId,Status:status,StatusMessage:statusMessage,VpcId:vpcId}'`
- Get endpoint details: `aws deadline get-license-endpoint --license-endpoint-id <id>`
- Verify VPC and subnet configuration for the endpoint

SHOULD:
- Check security group allows traffic to license server port
- Verify DNS resolution for license server hostname from worker
- Test network connectivity from worker to license endpoint

MAY:
- Check license server availability independently
- Review license server logs for checkout failures

## Phase 2 — Remediate

MUST:
- Fix VPC/subnet configuration for license endpoint
- Ensure security groups allow traffic on license server port (typically 27000-27009)
- Verify license server is running and has available licenses

SHOULD:
- Configure DNS resolution for license server hostname
- Set up license endpoint in same VPC as fleet workers

MAY:
- Configure redundant license servers for high availability
- Set up license usage monitoring

## Safety Ratings

```
safety_ratings:
  - "list-license-endpoints: GREEN — read-only endpoint enumeration"
  - "get-license-endpoint: GREEN — read-only endpoint details"
  - "create-license-endpoint: YELLOW — creates VPC endpoint resource"
  - "delete-license-endpoint: RED — removes license connectivity, breaks licensed renders"
  - "Security group changes: YELLOW — modifies network access"
```

## Escalation Conditions

- License server is down requiring vendor support
- VPC peering or Transit Gateway needed for cross-VPC license access
- License count exhausted requiring procurement
- License endpoint creation fails due to VPC endpoint limits

## Data Sensitivity

- License endpoint configuration: MEDIUM — reveals VPC and network architecture
- License server address: MEDIUM — infrastructure endpoint
- License keys/credentials: CRITICAL — must never be exposed
- License usage data: MEDIUM — reveals software usage patterns

## Prohibited Actions

- NEVER expose license server ports to the public internet
- NEVER share license server credentials or keys
- NEVER delete license endpoints during active render sessions using licensed software
- NEVER modify license server configuration without vendor guidance

## Phase 3 — Rollback

1. If license endpoint was deleted, recreate with same VPC/subnet configuration
2. If security group was modified, restore previous rules
3. If DNS configuration was changed, revert DNS records
4. Verify license checkout works from a worker after fix

## Common Issues

- symptoms: "License checkout fails with connection timeout"
  diagnosis: "Security group blocking traffic to license server port or network route missing."
  resolution: "Allow outbound traffic to license server IP:port in security group. Verify route table."

- symptoms: "License endpoint status shows CREATE_FAILED"
  diagnosis: "VPC endpoint creation failed due to subnet or security group issues."
  resolution: "Verify subnet has available IPs. Check security group exists in the same VPC."

## Output Format

```yaml
root_cause: "license_endpoint — <specific_cause>"
evidence:
  - type: endpoint_detail
    content: "<license endpoint configuration and status>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint connectivity"
  long_term: "Implement license server monitoring and redundancy"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
