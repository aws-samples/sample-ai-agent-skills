---
title: "D2 — Hybrid Activation Errors"
description: "Diagnose hybrid activation registration and connectivity errors for on-premises nodes"
status: active
severity: HIGH
triggers: ["hybrid node not registering", "on-premises SSM failed", "mi- instance errors"]
owner: devops-agent
objective: "Resolve hybrid activation registration and connectivity issues"
context: "Hybrid activations register on-premises servers as mi-* managed instances. They require SSM Agent installation, activation code/ID, and outbound HTTPS connectivity to SSM endpoints."
---

## Phase 1 — Triage

MUST:
- Check hybrid instance status: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<mi-instance-id>`
- Verify activation is valid: `aws ssm describe-activations --filters FilterKey=DefaultInstanceName,FilterValues=<name>`
- Check connection status: `aws ssm get-connection-status --target <mi-instance-id>`

SHOULD:
- Verify the on-premises server can reach SSM endpoints: `nslookup ssm.<region>.amazonaws.com`
- Check SSM Agent logs on the on-premises server: `/var/log/amazon/ssm/amazon-ssm-agent.log`

MAY:
- Test HTTPS connectivity to SSM endpoints from the on-premises network
- Review proxy configuration if the server uses a proxy for internet access

## Phase 2 — Remediate

MUST:
- Install SSM Agent on the on-premises server if not installed
- Register with valid activation: `amazon-ssm-agent -register -code <code> -id <id> -region <region>`
- Ensure outbound HTTPS (443) to SSM endpoints is allowed through firewalls

SHOULD:
- Configure proxy settings in SSM Agent if required: edit `/etc/amazon/ssm/seelog.xml`
- Verify DNS resolution for SSM endpoints from the on-premises network

MAY:
- Set up AWS PrivateLink with VPN/Direct Connect for private connectivity
- Configure SSM Agent to use a specific endpoint URL

## Common Issues

- symptoms: "SSM Agent registration fails with 'unable to register' error"
  diagnosis: "Activation expired, registration limit reached, or network connectivity issue."
  resolution: "Create new activation if expired. Check registration limit. Verify HTTPS connectivity to SSM endpoints."

- symptoms: "Hybrid node registers but shows Connection Lost after minutes"
  diagnosis: "Intermittent network connectivity or firewall blocking SSM heartbeat."
  resolution: "Ensure persistent HTTPS connectivity to ssm, ssmmessages, and ec2messages endpoints."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only query"
  - "describe-activations: GREEN — read-only activation listing"
  - "amazon-ssm-agent -register: YELLOW — registers server with AWS account"
  - "deregister-managed-instance: RED — removes instance from SSM management"
```

## Escalation Conditions

- Multiple on-premises servers fail to register simultaneously
- Registration succeeds but all hybrid nodes show Connection Lost
- Network team cannot open required firewall rules for SSM endpoints
- Hybrid nodes in air-gapped environments require special connectivity solutions

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata only
- SSM Agent registration: HIGH — associates on-premises server with AWS account
- SSM Agent logs: MEDIUM — may contain activation codes and endpoint details
- Network connectivity tests: LOW — reveals endpoint reachability only

## Prohibited Actions

- NEVER store activation codes in version control or shared documentation
- NEVER deregister hybrid instances without confirming they are no longer needed
- NEVER modify on-premises firewall rules without network team approval
- NEVER install SSM Agent from unofficial sources on on-premises servers

## Phase 3 — Rollback

1. If a server was registered incorrectly, deregister it: `aws ssm deregister-managed-instance --instance-id <mi-id>`
2. If SSM Agent was installed and needs removal, uninstall via package manager
3. If firewall rules were changed, coordinate with network team to revert
4. If proxy configuration was modified, restore original SSM Agent config
5. Document all changes to on-premises infrastructure

## Output Format

```yaml
root_cause: "hybrid_activation — <specific_cause>"
evidence:
  - type: hybrid_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix hybrid registration"
  long_term: "Automate hybrid node lifecycle with configuration management"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
