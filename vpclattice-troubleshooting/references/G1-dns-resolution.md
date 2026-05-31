---
title: "G1 — DNS Resolution Failures"
description: "Diagnose VPC Lattice service DNS resolution issues"
status: active
severity: HIGH
triggers:
  - "DNS not resolving"
  - "NXDOMAIN"
  - "cannot resolve service"
  - "DNS lookup failed"
owner: devops-agent
objective: "Resolve DNS resolution failures for VPC Lattice services"
context: "VPC Lattice services get a generated DNS name that resolves to link-local addresses. DNS resolution only works from VPCs associated with the service network. Custom domains require Route 53 alias or CNAME records."
---

## Phase 1 — Triage

MUST:
- Get service DNS name: `aws vpc-lattice get-service --service-identifier <id> --query 'dnsEntry'`
- Verify VPC is associated with service network: `aws vpc-lattice list-service-network-vpc-associations --service-network-identifier <id>`
- Verify association is ACTIVE
- Test DNS from within the VPC: `nslookup <service-dns-name>`

SHOULD:
- Check VPC DNS settings (enableDnsSupport and enableDnsHostnames must be true)
- Verify no custom DNS server overriding VPC DNS resolution
- Check if using custom domain name (requires Route 53 configuration)

## Phase 2 — Remediate

MUST:
- Associate VPC with service network if not associated
- Enable VPC DNS support: `aws ec2 modify-vpc-attribute --vpc-id <id> --enable-dns-support`
- For custom domains: create Route 53 alias record pointing to generated DNS name

SHOULD:
- Verify DNS resolution returns link-local address (169.254.171.x)
- Test connectivity after DNS resolves

## Output Format

```yaml
root_cause: "dns-resolution — <specific_cause>"
evidence:
  - type: service_dns
    content: "<generated DNS name>"
  - type: vpc_association
    content: "<association status>"
  - type: dns_test
    content: "<nslookup result>"
severity: HIGH
mitigation:
  immediate: "Associate VPC or fix DNS settings"
  long_term: "Use custom domains with Route 53 for stable naming"
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

- Service network configurations reveal microservice architecture
- Auth policies contain inter-service access control rules
- Access logs reveal service-to-service communication patterns
- Classification: INTERNAL - service mesh topology and access patterns

## Prohibited Actions

- NEVER delete service networks with active service associations
- NEVER modify auth policies that could disrupt inter-service communication
- NEVER remove target groups serving production traffic
- NEVER change listener rules without testing routing behavior first

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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
