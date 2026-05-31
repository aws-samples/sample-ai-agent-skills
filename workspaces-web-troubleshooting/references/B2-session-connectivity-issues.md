---
title: "B2 — Session Connectivity Issues"
description: "Diagnose WorkSpaces Web session connectivity and website access issues"
status: active
severity: MEDIUM
triggers: ["cannot access website", "site not loading", "connection refused", "timeout in session", "ERR_CONNECTION"]
owner: devops-agent
objective: "Resolve connectivity issues within browser sessions to target websites"
context: "Browser sessions route traffic through the configured VPC. Connectivity depends on VPC routing, security groups, NACLs, DNS resolution, and target site availability."
---

## Phase 1 — Triage

MUST:
- Check network settings: `aws workspaces-web get-network-settings --network-settings-arn <arn>`
- Verify security group outbound rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`
- Check VPC route tables for the session subnets: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Verify DNS resolution within the VPC (check VPC DNS settings)
- Check if the target site is accessible from the VPC (not a session-specific issue)
- Verify NACLs are not blocking traffic: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

MAY:
- Test connectivity from an EC2 instance in the same VPC and subnets
- Check if the target site requires specific IP allowlisting
- Verify proxy configuration in browser settings if applicable

## Phase 2 — Remediate

MUST:
- Add security group rules to allow outbound traffic to the target site
- Ensure VPC route tables include routes for the target destination
- Verify DNS resolution for the target hostname within the VPC

SHOULD:
- Configure VPC endpoints for AWS service access if needed
- Add NAT gateway if sessions need internet access and subnets are private
- Update NACLs if they are blocking required traffic

MAY:
- Configure browser proxy settings for sites requiring proxy access
- Set up VPC peering or Transit Gateway for cross-VPC access
- Add trust store certificates for internal sites with private CAs

## Safety Ratings

```
safety_ratings:
  - "get-network-settings: GREEN — read-only network configuration"
  - "describe-security-groups: GREEN — read-only security rules"
  - "describe-route-tables: GREEN — read-only routing information"
  - "modify security group rules: RED — affects network security posture"
  - "modify route tables: RED — affects VPC routing for all resources"
```

## Escalation Conditions

- Connectivity issue affects all sites (not just specific targets)
- VPC routing changes required that affect other workloads
- Target site requires IP allowlisting and session IPs are dynamic
- Cross-account or cross-VPC networking changes needed

## Data Sensitivity

- `aws workspaces-web get-network-settings`: MEDIUM — reveals VPC configuration
- `aws ec2 describe-security-groups`: MEDIUM — reveals security rules
- `aws ec2 describe-route-tables`: MEDIUM — reveals network routing
- `aws ec2 describe-network-acls`: MEDIUM — reveals network access controls

## Prohibited Actions

- NEVER open all outbound traffic (0.0.0.0/0 all ports) as a troubleshooting shortcut
- NEVER modify VPC routing without understanding the impact on other resources
- NEVER bypass proxy requirements by removing proxy configuration
- NEVER modify NACLs without documenting the changes for the network team

## Phase 3 — Rollback

1. Revert security group rule changes if they did not resolve the issue
2. Restore previous route table entries if routing changes caused problems
3. Revert NACL changes if they affected other resources
4. Remove VPC peering or Transit Gateway attachments if they were created in error
5. Document all network changes and their outcomes

## Common Issues

- symptoms: "Internal website shows ERR_CERT_AUTHORITY_INVALID in session"
  diagnosis: "Internal site uses a private CA certificate not in the portal trust store."
  resolution: "Upload the CA certificate to a trust store and associate it with the portal."

## Output Format

```yaml
root_cause: "session_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix network path to target site"
  long_term: "Document network requirements and implement connectivity monitoring"
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
