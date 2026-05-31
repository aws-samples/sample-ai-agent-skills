---
title: "G2 — Security Group Issues"
description: "Diagnose security group configuration problems for Verified Access"
status: active
severity: HIGH
triggers:
  - "security group blocking"
  - "traffic blocked"
  - "security group configuration"
  - "inbound rules"
owner: devops-agent
objective: "Identify and resolve security group issues for Verified Access endpoints"
context: "Verified Access endpoints have security groups controlling network access. The endpoint security group must allow inbound HTTPS (443) and outbound to the backend. The backend security group must allow inbound from the endpoint security group. Both must be correctly configured."
---

## Phase 1 — Triage

MUST:
- Check endpoint security groups: `aws ec2 describe-verified-access-endpoints --verified-access-endpoint-ids <endpoint-id> --query 'VerifiedAccessEndpoints[*].SecurityGroupIds'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify inbound HTTPS (443) is allowed on endpoint security group
- Verify backend security group allows traffic from endpoint security group

SHOULD:
- Check outbound rules on endpoint security group
- Verify no overly restrictive rules
- Check VPC Flow Logs for rejected traffic

MAY:
- Compare working vs non-working endpoint security groups
- Review security group change history in CloudTrail

## Phase 2 — Remediate

MUST:
- Allow inbound HTTPS on endpoint: `aws ec2 authorize-security-group-ingress --group-id <endpoint-sg> --protocol tcp --port 443 --cidr 0.0.0.0/0`
- Allow endpoint to reach backend: add outbound rule to endpoint SG and inbound rule to backend SG referencing endpoint SG
- Verify connectivity after changes

SHOULD:
- Use security group references instead of CIDR for endpoint-to-backend rules
- Restrict inbound to known IP ranges where possible

MAY:
- Implement security group change monitoring
- Use AWS Firewall Manager for centralized SG management

## Common Issues

- symptoms: "Users cannot reach endpoint"
  diagnosis: "Endpoint security group missing inbound HTTPS (443)."
  resolution: "Add inbound rule for TCP 443."

- symptoms: "Endpoint reaches IdP but not backend"
  diagnosis: "Backend security group doesn't allow traffic from endpoint SG."
  resolution: "Add inbound rule on backend SG referencing endpoint SG."

- symptoms: "Intermittent connectivity"
  diagnosis: "Security group rules too restrictive or ephemeral port range blocked."
  resolution: "Check outbound rules. Ensure return traffic is allowed."

## Output Format

```yaml
root_cause: "security_groups — <specific_cause>"
evidence:
  - type: endpoint_sg
    content: "<endpoint security group rules>"
  - type: backend_sg
    content: "<backend security group rules>"
  - type: flow_logs
    content: "<VPC Flow Log entries>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules"
  long_term: "Document security group requirements and monitor changes"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
