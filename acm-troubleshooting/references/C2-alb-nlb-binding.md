---
title: "C2 — ALB/NLB Certificate Binding"
description: "Diagnose certificate issues with Application Load Balancers and Network Load Balancers"
status: active
severity: HIGH
triggers:
  - "ALB certificate"
  - "NLB certificate"
  - "load balancer SSL"
  - "listener certificate"
  - "HTTPS listener"
  - "TLS listener"
owner: devops-agent
objective: "Ensure ACM certificates are correctly bound to ALB/NLB HTTPS/TLS listeners"
context: "ALB and NLB support ACM certificates on HTTPS/TLS listeners. The certificate must be in the SAME region as the load balancer. ALB supports SNI with multiple certificates per listener. NLB TLS listeners also support ACM certificates. Common issues: wrong region, certificate not ISSUED, listener misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check the load balancer's listeners: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`
- Check the certificate on the HTTPS/TLS listener: `aws elbv2 describe-listener-certificates --listener-arn <listener-arn>`
- Verify the certificate status: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,NotAfter:NotAfter,DomainName:DomainName}'`
- Confirm the certificate is in the same region as the load balancer

SHOULD:
- Check if the certificate domain matches the DNS name used to access the ALB/NLB
- Verify the listener protocol is HTTPS (ALB) or TLS (NLB), not HTTP/TCP
- Check the SSL policy on the listener: `aws elbv2 describe-listeners --listener-arn <listener-arn> --query 'Listeners[].SslPolicy'`
- For SNI: list all certificates on the listener: `aws elbv2 describe-listener-certificates --listener-arn <listener-arn>`

MAY:
- Check security group allows inbound on port 443
- Verify target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check for certificate expiry warnings

## Phase 2 — Remediate

MUST:
- If no certificate on listener: add one: `aws elbv2 add-listener-certificates --listener-arn <listener-arn> --certificates CertificateArn=<cert-arn>`
- If certificate is expired or invalid: replace with a valid certificate: `aws elbv2 modify-listener --listener-arn <listener-arn> --certificates CertificateArn=<new-cert-arn>`
- If certificate is in wrong region: request a new certificate in the load balancer's region

SHOULD:
- Set the default certificate on the listener for the primary domain
- Add additional certificates for SNI support on ALB
- Update the SSL policy to a current recommended policy (e.g., ELBSecurityPolicy-TLS13-1-2-2021-06)

MAY:
- Automate certificate rotation with CloudFormation or Terraform
- Set up CloudWatch alarms for certificate expiry

## Common Issues

- symptoms: "HTTPS listener returns certificate error in browser"
  diagnosis: "Certificate domain doesn't match the URL, or certificate is expired."
  resolution: "Verify certificate SANs cover the domain. Check NotAfter for expiry."

- symptoms: "Cannot add certificate to listener"
  diagnosis: "Certificate is not in ISSUED status or is in a different region."
  resolution: "Ensure certificate is ISSUED and in the same region as the load balancer."

- symptoms: "SNI not working — wrong certificate served"
  diagnosis: "Default certificate is served when no SNI match. Additional certificates need to be added."
  resolution: "Add certificates for each domain using add-listener-certificates. Set the correct default."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add certificate to listener | GREEN | Additive action, does not remove existing certificates |
| Modify listener default certificate | YELLOW | Changes active certificate; may cause brief TLS renegotiation |
| Update SSL policy | YELLOW | May drop connections using deprecated cipher suites |
| Remove old certificate from listener | YELLOW | Removes SNI option; ensure replacement is in place |

## Escalation Conditions

- Production load balancer serving certificate errors — escalate immediately to service team
- SSL policy change may affect client compatibility — escalate to application team for testing
- Certificate replacement on high-traffic listener — escalate to operations team for change window
- Multiple listeners across multiple load balancers affected — escalate to infrastructure team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws elbv2 describe-listeners` | MEDIUM | Exposes listener configuration and certificate ARNs |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names and service associations |
| `aws elbv2 describe-target-health` | MEDIUM | Exposes backend target health status |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** remove the only certificate from a listener without adding a replacement first

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Added certificate to listener | Remove the certificate: `aws elbv2 remove-listener-certificates` |
| Modified default certificate | Revert to previous default: `aws elbv2 modify-listener --certificates CertificateArn=<old-arn>` |
| Updated SSL policy | Revert to previous policy: `aws elbv2 modify-listener --ssl-policy <old-policy>` |

## Output Format

```yaml
root_cause: "alb_nlb_certificate — <specific_cause>"
evidence:
  - type: listener_config
    content: "<describe-listeners output>"
  - type: certificate_status
    content: "<describe-certificate output>"
severity: HIGH
mitigation:
  immediate: "Bind correct certificate to the listener"
  long_term: "Automate certificate management with IaC and set up expiry monitoring"
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
  - "NEVER suggest disabling certificate transparency"
  - "NEVER suggest using expired certificates"
  - "NEVER suggest self-signed certs for production"
