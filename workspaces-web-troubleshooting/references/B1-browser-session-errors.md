---
title: "B1 — Browser Session Errors"
description: "Diagnose WorkSpaces Web browser session launch and runtime errors"
status: active
severity: HIGH
triggers: ["session failed to start", "browser error", "session crash", "blank screen", "session timeout"]
owner: devops-agent
objective: "Resolve browser session launch failures and runtime errors"
context: "Browser sessions are AWS-managed Chrome instances. Sessions require a working portal with ACTIVE status, valid IdP authentication, and functional network settings."
---

## Phase 1 — Triage

MUST:
- Verify portal status is ACTIVE: `aws workspaces-web get-portal --portal-arn <portal-arn> --query 'portal.portalStatus'`
- Check network settings for the portal: `aws workspaces-web get-network-settings --network-settings-arn <arn>`
- Verify the user successfully authenticated via SAML (check IdP logs)

SHOULD:
- Check if the issue affects all users or specific users
- Verify VPC subnets have available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{SubnetId:SubnetId,AvailableIpAddressCount:AvailableIpAddressCount}'`
- Check security group rules allow outbound traffic: `aws ec2 describe-security-groups --group-ids <sg-id>`

MAY:
- Check CloudTrail for session-related events
- Review user access logging if enabled: check the associated Kinesis Data Stream
- Check AWS Health Dashboard for WorkSpaces Web service issues

## Phase 2 — Remediate

MUST:
- Ensure portal status is ACTIVE with all required associations
- Verify VPC subnets have sufficient available IP addresses
- Ensure security groups allow outbound HTTPS (443) for browser session traffic

SHOULD:
- Verify NAT gateway or internet gateway is configured for internet access from session subnets
- Check that DNS resolution works within the VPC
- Verify the browser settings policy does not block the target URL

MAY:
- Increase subnet size if IP exhaustion is the issue
- Add additional subnets in more availability zones
- Configure browser settings to allow specific URLs

## Safety Ratings

```
safety_ratings:
  - "get-portal: GREEN — read-only portal status"
  - "get-network-settings: GREEN — read-only network configuration"
  - "describe-subnets: GREEN — read-only subnet details"
  - "describe-security-groups: GREEN — read-only security group rules"
  - "modify security groups: RED — affects network security posture"
```

## Escalation Conditions

- All users unable to start sessions despite correct portal configuration
- Sessions start but crash immediately with no error details
- Session failures correlate with specific times suggesting capacity issues
- VPC or subnet changes required that affect other workloads

## Data Sensitivity

- `aws workspaces-web get-portal`: LOW — returns portal metadata
- `aws ec2 describe-subnets`: LOW — returns subnet metadata
- `aws ec2 describe-security-groups`: MEDIUM — reveals network security rules
- User access logs (Kinesis): HIGH — contains user session activity data

## Prohibited Actions

- NEVER suggest SSH or console access to browser session hosts — they are AWS-managed
- NEVER modify security groups to allow all inbound traffic for debugging
- NEVER access user session content or recordings without authorization
- NEVER modify VPC routing during active user sessions without a maintenance window

## Phase 3 — Rollback

1. Revert security group changes if they did not resolve the issue or introduced security gaps
2. Restore previous network settings association if changes worsened the situation
3. Revert browser settings if policy changes caused session failures
4. Verify sessions are working for a test user before declaring the issue resolved
5. Document the session error timeline and resolution

## Common Issues

- symptoms: "Users see blank screen after SAML authentication"
  diagnosis: "VPC subnets have no route to the internet — sessions cannot load web content."
  resolution: "Ensure subnets have a NAT gateway route for internet access."

## Output Format

```yaml
root_cause: "browser_session — <specific_cause>"
evidence:
  - type: session_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix session launch issue"
  long_term: "Implement session monitoring and capacity planning"
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
