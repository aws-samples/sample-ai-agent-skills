---
title: "D2 — Security Group Issues"
description: "Diagnose EMR security group configuration problems"
status: active
severity: HIGH
triggers:
  - "security group error"
  - "connectivity between nodes"
  - "EMR managed security groups"
  - "port blocked"
owner: devops-agent
objective: "Identify and resolve EMR security group issues"
context: "EMR uses managed security groups for master, core/task, and service access. Managed SGs allow inter-node communication. Additional SGs can be specified. Incorrect SG configuration blocks node communication, service access, or external connectivity."
---

## Phase 1 — Triage

MUST:
- Check EMR managed security groups: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.Ec2InstanceAttributes.{MasterSG:EmrManagedMasterSecurityGroup,SlaveSG:EmrManagedSlaveSecurityGroup,ServiceSG:ServiceAccessSecurityGroup}'`
- Verify security group rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Check for self-referencing rules (inter-node communication)
- Verify service access security group allows EMR service communication

SHOULD:
- Check additional security groups for restrictive rules
- Verify port 8443 is open for EMR service access
- Check for NACL rules that may override SG rules
- Verify SSH access (port 22) if needed for debugging

MAY:
- Check VPC flow logs for blocked traffic
- Review security group change history in CloudTrail

## Phase 2 — Remediate

MUST:
- Ensure EMR managed security groups allow inter-node communication
- Verify service access security group allows port 8443 from EMR service
- Fix any restrictive rules blocking required ports

SHOULD:
- Use EMR-managed security groups (let EMR create them)
- Add custom rules via additional security groups, not by modifying managed SGs
- Document required ports for EMR communication

MAY:
- Implement security group monitoring
- Create security group templates for EMR

## Common Issues

- symptoms: "Nodes cannot communicate with each other"
  diagnosis: "Managed security groups modified or missing self-referencing rules."
  resolution: "Restore EMR managed security group rules. Ensure all-traffic self-reference."

- symptoms: "EMR service cannot communicate with cluster"
  diagnosis: "Service access security group missing or port 8443 blocked."
  resolution: "Ensure service access SG allows inbound 8443 from EMR service."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Restore EMR managed security group rules | YELLOW | Changes security group rules — verify no unintended access |
| Add custom rules via additional security groups | GREEN | Additive change — non-destructive |
| Open port 8443 for EMR service access | YELLOW | Opens network port — verify source restriction |
| Open port 22 for SSH debugging | YELLOW | Opens SSH access — restrict to specific IPs |
| Implement security group monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Security group changes affect other resources sharing the same groups
- Port changes require security team approval

## Data Sensitivity

- **Classification: HIGH**
- Security group rules reveal network access boundaries and trust relationships
- Port configurations expose service communication patterns
- VPC flow logs may contain connection metadata for sensitive services
- Security group change history reveals operational patterns

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest opening all inbound traffic (0.0.0.0/0) on EMR security groups
- **NEVER** suggest modifying EMR-managed security groups directly — use additional security groups

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Restored managed security group rules | Revert rules if they cause unintended access |
| Added custom security group rules | Remove added rules from additional security groups |
| Opened port 8443 for EMR service | Remove the rule if service access is no longer needed |
| Opened port 22 for SSH | Remove SSH access rule after debugging is complete |

## Output Format

```yaml
root_cause: "security_groups — <specific_cause>"
evidence:
  - type: security_groups
    content: "<security group rules>"
  - type: connectivity
    content: "<connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules"
  long_term: "Use EMR-managed security groups and document required ports"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
