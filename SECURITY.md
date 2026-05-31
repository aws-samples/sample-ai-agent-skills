# Security Framework for DevOps Agent Skills

## Important Disclaimer

These skills are reference resources designed for demonstration and educational purposes. They contain structured troubleshooting guidance that influences AI agent behavior against live cloud infrastructure. While the skills themselves contain no executable code, they direct an agent that does have execution capabilities.

```
Skill Content (markdown/yaml) → AI Agent (interprets) → Cloud APIs (executes)
```

There is no checksum verification, GPG signing, or tamper detection on skill files. Organizations deploying these skills in production should implement their own integrity verification.

## Threat Model

| Risk | Description | Mitigation |
|------|-------------|------------|
| Tampered skills | Modified skill files could guide agent toward destructive actions | Implement file integrity monitoring, use version control |
| Overly broad remediation | Agent interprets "fix the policy" as granting wildcard permissions | Prohibited actions list in each skill |
| Sensitive data exposure | Triage commands return IAM policies, account IDs, key policies | Data sensitivity classifications |
| No rollback guidance | Failed remediation with no revert path | Rollback sections in runbooks |
| Partial investigation cleanup | Interrupted investigations leave temporary resources | Cleanup notes for resource-creating skills |

## Prohibited Actions (Global)

The following actions MUST NEVER be suggested by the agent as remediation, regardless of the skill or scenario:

### IAM
- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any IAM policy fix
- NEVER suggest `"Effect": "Allow", "Action": "*", "Resource": "*"` as a troubleshooting step
- NEVER suggest granting `AdministratorAccess` to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied errors

### S3
- NEVER suggest disabling S3 Block Public Access as a remediation step
- NEVER suggest `"Principal": "*"` in a bucket policy without restrictive Condition keys
- NEVER suggest removing bucket encryption to fix access issues

### EKS
- NEVER suggest adding `system:masters` group to aws-auth for troubleshooting access
- NEVER suggest disabling RBAC or Pod Security Standards
- NEVER suggest running containers as root to fix permission issues

### KMS
- NEVER suggest `"kms:*"` on a key policy as a fix
- NEVER suggest disabling key rotation to resolve issues
- NEVER suggest making a KMS key policy public

### Networking
- NEVER suggest `0.0.0.0/0` inbound rules on security groups as a fix
- NEVER suggest disabling NACLs to troubleshoot connectivity
- NEVER suggest removing all firewall rules to test connectivity

### CloudFormation
- NEVER leave a stack policy open after temporary modification
- NEVER suggest deleting a stack to "fix" an UPDATE_ROLLBACK_FAILED without exploring ContinueUpdateRollback first

### General
- NEVER suggest disabling logging, auditing, or monitoring as a troubleshooting step
- NEVER suggest using root account credentials for troubleshooting
- NEVER suggest sharing access keys or credentials in any form

## Safety Classification System

Every remediation action in Phase 2 of runbooks should be classified:

| Rating | Description | Examples |
|--------|-------------|----------|
| GREEN | Read-only, no state changes | describe-*, get-*, list-*, kubectl get |
| YELLOW | State-changing but recoverable | Modify security group, update parameter, restart service |
| RED | Destructive or irreversible | Delete resource, terminate instance, remove encryption, modify key policy |

## Escalation Conditions

The agent MUST escalate (stop and ask a human) when:

1. Remediation requires modifying IAM policies in a production account
2. Remediation requires disabling a security control (even temporarily)
3. Remediation requires modifying KMS key policies
4. Remediation requires editing aws-auth ConfigMap in a production EKS cluster
5. Remediation requires modifying Organization SCPs
6. Remediation requires changes to CloudTrail or audit logging
7. Remediation requires modifying encryption settings
8. The root cause cannot be identified after 3 hypothesis pivots
9. The blast radius affects more than one account or region
10. The issue involves data loss or potential data exposure

## Data Sensitivity Classifications

### HIGH sensitivity (redact from stored output)
- Full IAM policies and key policies
- Account IDs and organization structure
- IP addresses and network configurations
- Security group rules and NACL rules
- aws-auth ConfigMap contents
- API keys, tokens, and credentials
- GuardDuty findings with threat indicators
- CloudTrail events with user identities

### MEDIUM sensitivity (summarize, don't reproduce verbatim)
- Resource ARNs
- VPC and subnet configurations
- Parameter group settings
- Service quotas and limits

### LOW sensitivity (safe to include in output)
- Resource status and health
- CloudWatch metrics (aggregated)
- Error messages and status codes
- Configuration settings (non-security)

## Rollback Guidance

Every Phase 2 remediation that modifies state MUST include:

1. **Pre-change snapshot**: Document the current state before modification
2. **Verification step**: Confirm the change resolved the issue
3. **Rollback procedure**: Explicit steps to revert if the change made things worse
4. **Cleanup**: Remove any temporary resources created during investigation

## Skill Versioning

Each SKILL.md should include a version field in the frontmatter:

```yaml
version: "1.0.0"
last_updated: "2025-04-09"
```

This enables tracking which skill version was active during an investigation.

## Remediation Hallucination Patterns

In addition to diagnostic hallucination patterns, each skill should include remediation-side patterns:

```yaml
remediation_hallucinations:
  - id: R1-overly-broad-fix
    trigger: "access denied"
    dangerous_suggestion: "Add Action: * to the IAM policy"
    correct_approach: "Identify the specific denied action from CloudTrail and grant only that action on the specific resource"
    
  - id: R2-disable-security-control
    trigger: "blocked by security"
    dangerous_suggestion: "Disable the security control to unblock"
    correct_approach: "Add a specific exception for the legitimate use case while keeping the control active"
```

## Cleanup Guidance

Skills that create temporary resources during investigation must document cleanup:

| Resource Type | Created During | Cleanup Action |
|--------------|----------------|----------------|
| SSM commands | EC2 triage | Commands auto-expire, but check `ssm list-commands` |
| API keys | AMG/AMP setup | Delete temporary keys after investigation |
| Console connections | OCI compute | Connections expire in 24 hours |
| VPC Reachability Analyzer | Network triage | Delete analysis after review |
| Config rule evaluations | Compliance check | No cleanup needed (read-only) |
| CloudFormation change sets | Stack debugging | Delete unused change sets |

## Recommendations for Production Deployment

1. **Integrity verification**: Hash all skill files and verify before loading
2. **Access control**: Restrict who can upload/modify skills
3. **Audit logging**: Log which skills are activated and what commands the agent executes
4. **Least privilege**: Run the agent with minimum required permissions
5. **Approval gates**: Require human approval for YELLOW/RED remediation actions
6. **Output sanitization**: Redact sensitive data from investigation reports
7. **Version pinning**: Pin skill versions and review changes before updating
8. **Testing**: Test skills in non-production environments before deploying to production
