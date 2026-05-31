---
title: "B1 — Certificate-Based Authentication"
description: "Diagnose VPN certificate-based authentication failures using ACM Private CA"
status: active
severity: HIGH
triggers: ["certificate VPN auth failed", "ACM PCA VPN error", "VPN certificate expired", "certificate-based VPN not connecting"]
owner: devops-agent
objective: "Resolve VPN certificate-based authentication issues involving ACM Private CA"
context: "Certificate-based VPN authentication uses certificates from ACM Private CA instead of pre-shared keys. The PCA must be in the same region as the VPN connection."
---

## Phase 1 — Triage

MUST:
- Check VPN connection authentication type: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Options.TunnelOptions[*].{TunnelInsideCidr:TunnelInsideCidr}'`
- List ACM PCA certificate authorities: `aws acm-pca list-certificate-authorities --query 'CertificateAuthorities[*].{Arn:Arn,Status:Status,Type:Type}'`
- Verify the PCA is ACTIVE and in the same region as the VPN connection

SHOULD:
- Check certificate expiry: `aws acm list-certificates --query 'CertificateSummaryList[*].{Arn:CertificateArn,DomainName:DomainName,Status:Status,NotAfter:NotAfter}'`
- Verify the certificate was issued by the correct PCA
- Check VPN logs for IKE authentication errors: `aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --filter-pattern "authentication" --limit 20`

MAY:
- Verify the certificate chain is complete (root CA → subordinate CA → end-entity)
- Check if the customer gateway supports certificate-based authentication

## Phase 2 — Remediate

MUST:
- If PCA is not ACTIVE: activate or create a new PCA
- If certificate expired: issue a new certificate from the PCA and update the VPN connection
- Ensure the PCA is in the same region as the VPN connection

SHOULD:
- Set up certificate auto-renewal before expiry
- Verify the customer gateway is configured with the correct certificate and CA chain

MAY:
- Implement certificate rotation automation
- Set up CloudWatch alarms for certificate expiry

## Common Issues

- symptoms: "VPN tunnel fails IKE authentication with certificate error"
  diagnosis: "Certificate expired or issued by wrong PCA."
  resolution: "Issue a new certificate from the correct ACM PCA and update the VPN connection tunnel options."

- symptoms: "Cannot create VPN connection with certificate authentication"
  diagnosis: "ACM PCA not in the same region or PCA is not ACTIVE."
  resolution: "Create or activate an ACM PCA in the same region as the VPN connection."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN details"
  - "acm-pca list-certificate-authorities: GREEN — read-only PCA listing"
  - "acm list-certificates: GREEN — read-only certificate listing"
  - "acm-pca issue-certificate: YELLOW — issues new certificate"
  - "modify-vpn-tunnel-options: RED — changes tunnel authentication, causes tunnel restart"
```

## Escalation Conditions

- ACM PCA is in DISABLED or DELETED state and cannot be recovered
- Certificate chain is broken and requires PCA hierarchy reconstruction
- Customer gateway vendor does not support the certificate format issued by ACM PCA
- Certificate renewal automation fails and certificates expire in production

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration
- `aws acm-pca list-certificate-authorities`: MEDIUM — reveals PCA infrastructure
- `aws acm list-certificates`: MEDIUM — reveals certificate details
- VPN logs: MEDIUM — reveals IKE negotiation details

## Prohibited Actions

- NEVER delete an ACM PCA that has issued certificates for active VPN connections
- NEVER share private keys or certificate material in logs or tickets
- NEVER use public ACM certificates for VPN authentication — only ACM PCA certificates work
- NEVER modify tunnel authentication options on both tunnels simultaneously

## Phase 3 — Rollback

1. If certificate changes caused tunnel failures, revert to the previous certificate
2. If PCA was modified, restore PCA configuration
3. Verify tunnel authentication is working: check VgwTelemetry for UP status
4. If reverting to pre-shared key authentication is needed, modify tunnel options one tunnel at a time
5. Document the rollback and schedule certificate renewal

## Output Format

```yaml
root_cause: "certificate_auth — <specific_cause>"
evidence:
  - type: cert_error
    content: "<certificate or PCA details>"
severity: HIGH
mitigation:
  immediate: "Fix certificate authentication"
  long_term: "Implement certificate auto-renewal and monitoring"
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
