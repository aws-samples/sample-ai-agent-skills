---
title: "A3 — Hybrid Activation"
description: "Diagnose hybrid activation failures for on-premises servers"
status: active
severity: HIGH
triggers:
  - "hybrid activation failed"
  - "on-premises registration"
  - "mi- instance"
  - "activation code expired"
owner: devops-agent
objective: "Identify why hybrid activation is failing and register on-premises servers"
context: "Hybrid activations allow on-premises servers and VMs to be managed by SSM. They require an activation code/ID, IAM service role, outbound HTTPS to SSM endpoints, and TLS 1.2. Registered instances get mi- prefix IDs."
---

## Phase 1 — Triage

MUST:
- List activations: `aws ssm describe-activations --query 'ActivationList[*].{Id:ActivationId,Limit:RegistrationLimit,Count:RegistrationsCount,Expired:Expired,Role:IamRole}'`
- Check if activation is expired or at registration limit
- Verify IAM service role exists: `aws iam get-role --role-name <service-role-name>`
- Check role trust policy allows ssm.amazonaws.com: `aws iam get-role --role-name <service-role-name> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- List hybrid managed instances: `aws ssm describe-instance-information --filters Key=ResourceType,Values=ManagedInstance`
- Verify role has AmazonSSMManagedInstanceCore policy: `aws iam list-attached-role-policies --role-name <service-role-name>`
- Check CloudTrail for activation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateActivation --max-results 10`

MAY:
- Create a new activation if current one is expired: `aws ssm create-activation --iam-role <role-name> --registration-limit 10 --default-instance-name "on-prem-server"`
- Verify on-premises server can reach SSM endpoints via HTTPS

## Phase 2 — Remediate

MUST:
- Create new activation if expired or limit reached
- Ensure IAM service role trusts ssm.amazonaws.com
- Verify outbound HTTPS connectivity from on-premises server

SHOULD:
- Set appropriate registration limits and expiration dates
- Use descriptive default instance names for identification
- Document activation codes securely (they cannot be retrieved after creation)

MAY:
- Automate activation creation and distribution
- Set up monitoring for activation expiration

## Common Issues

- symptoms: "Activation expired"
  diagnosis: "The activation code has passed its expiration date."
  resolution: "Create a new activation with appropriate expiration date."

- symptoms: "Registration limit reached"
  diagnosis: "Maximum number of instances already registered with this activation."
  resolution: "Create a new activation with higher registration limit."

- symptoms: "TLS handshake failure on on-premises server"
  diagnosis: "Server does not support TLS 1.2 or CA certificates are outdated."
  resolution: "Update TLS libraries and CA certificate bundle on the server."

## Output Format

```yaml
root_cause: "hybrid_activation — <specific_cause>"
evidence:
  - type: activation_status
    content: "<describe-activations output>"
  - type: iam_role
    content: "<service role and trust policy>"
severity: HIGH
mitigation:
  immediate: "Create new activation or fix IAM role trust policy"
  long_term: "Implement activation lifecycle management and monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
