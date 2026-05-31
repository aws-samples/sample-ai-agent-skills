---
title: "C1 — SSL/TLS Errors"
description: "Diagnose and resolve ELB SSL/TLS termination and negotiation errors"
status: active
severity: HIGH
triggers:
  - "SSL error"
  - "TLS handshake failed"
  - "SSL negotiation"
  - "HTTPS listener error"
  - "SSL policy"
owner: devops-agent
objective: "Resolve SSL/TLS errors on ELB listeners and restore secure traffic flow"
context: "ELB terminates SSL/TLS connections on HTTPS/TLS listeners using ACM or IAM certificates. SSL errors can arise from expired certificates, unsupported cipher suites, protocol version mismatches, or misconfigured SSL policies. ALB supports SSL termination on HTTPS listeners; NLB supports TLS listeners. The SSL security policy determines which protocols and ciphers are available for negotiation."
---

## Phase 1 — Triage

MUST:
- Check listener SSL configuration: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn> --query 'Listeners[*].{Port:Port,Protocol:Protocol,SslPolicy:SslPolicy,Certificates:Certificates}'`
- Verify certificate validity: `aws acm describe-certificate --certificate-arn <cert-arn> --query 'Certificate.{Status:Status,NotAfter:NotAfter,DomainName:DomainName,SubjectAlternativeNames:SubjectAlternativeNames}'`
- Check the SSL policy details: `aws elbv2 describe-ssl-policies --names <policy-name>`
- Test TLS connection: `openssl s_client -connect <lb-dns>:443 -servername <domain>`
- Check CloudWatch ClientTLSNegotiationErrorCount: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name ClientTLSNegotiationErrorCount --dimensions Name=LoadBalancer,Value=<lb-arn-suffix> --start-time <time> --end-time <time> --period 300 --statistics Sum`

SHOULD:
- Verify the certificate covers the requested domain (including SANs)
- Check if the certificate chain is complete
- Review the SSL policy for deprecated protocols (SSLv3, TLS 1.0)
- Verify the client supports the protocols/ciphers in the SSL policy

MAY:
- Check if HSTS headers are causing issues with certificate changes
- Review ALB access logs for TLS-related errors
- Test with different TLS versions: `openssl s_client -connect <lb-dns>:443 -tls1_2`

## Phase 2 — Remediate

MUST:
- If certificate expired: renew or replace the certificate on the listener: `aws elbv2 modify-listener --listener-arn <listener-arn> --certificates CertificateArn=<new-cert-arn>`
- If domain mismatch: use a certificate that covers the requested domain
- If protocol mismatch: update the SSL policy to support required protocols: `aws elbv2 modify-listener --listener-arn <listener-arn> --ssl-policy <policy-name>`
- If cipher suite mismatch: choose an SSL policy that includes ciphers supported by clients

SHOULD:
- Use the latest recommended SSL policy (e.g., ELBSecurityPolicy-TLS13-1-2-2021-06 for TLS 1.3)
- Enable ACM auto-renewal for certificates
- Add additional certificates for multiple domains using SNI: `aws elbv2 add-listener-certificates --listener-arn <listener-arn> --certificates CertificateArn=<cert-arn>`

MAY:
- Implement certificate monitoring and expiry alerts
- Set up automated certificate rotation
- Document the SSL policy requirements for compliance

## Common Issues

- symptoms: "ERR_SSL_VERSION_OR_CIPHER_MISMATCH in browser"
  diagnosis: "Client does not support any cipher suite in the ALB's SSL policy."
  resolution: "Use a less restrictive SSL policy or update the client. Check for legacy clients requiring TLS 1.0/1.1."

- symptoms: "Certificate error — domain name mismatch"
  diagnosis: "The certificate does not cover the domain being accessed."
  resolution: "Issue a new certificate with the correct domain or add the domain as a SAN."

- symptoms: "TLS handshake fails with 'certificate verify failed'"
  diagnosis: "Certificate chain is incomplete or the root CA is not trusted."
  resolution: "Ensure the full certificate chain is provided. Use ACM-issued certificates for automatic chain management."

- symptoms: "High ClientTLSNegotiationErrorCount"
  diagnosis: "Many clients failing TLS negotiation — likely SSL policy too restrictive or certificate issue."
  resolution: "Review the SSL policy. Check certificate validity. Analyze client TLS capabilities."

## Output Format

```yaml
root_cause: "SSL/TLS error — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<certificate details and validity>"
  - type: ssl_policy
    content: "<SSL policy protocols and ciphers>"
severity: HIGH
mitigation:
  immediate: "Fix certificate or SSL policy configuration"
  long_term: "Implement certificate monitoring and automated renewal"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/openssl test): GREEN — read-only"
  - "Phase 2 certificate replacement: YELLOW — briefly affects TLS connections"
  - "Phase 2 SSL policy changes: YELLOW — may break clients using deprecated protocols"

## Escalation Conditions

escalation_conditions:
  - "Certificate has expired and no replacement is available"
  - "SSL policy change would break a significant number of clients"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves compliance requirements (PCI DSS, HIPAA) for TLS versions"
  - "Certificate private key may have been compromised"

## Data Sensitivity

data_sensitivity:
  - command: "describe-certificate"
    sensitivity: MEDIUM
    contains: "Certificate details and domain names"
  - command: "openssl s_client"
    sensitivity: MEDIUM
    contains: "Certificate chain and TLS negotiation details"
  - command: "ALB access logs"
    sensitivity: MEDIUM
    contains: "Client IPs and TLS version information"

## Prohibited Actions

prohibited_actions:
  - "NEVER downgrade to SSLv3 or TLS 1.0 to fix compatibility issues without security review"
  - "NEVER use self-signed certificates on production load balancers"
  - "NEVER share or log certificate private keys"
  - "NEVER disable HTTPS listeners to work around TLS errors"

## Phase 3 — Rollback

- Before changing SSL policy, document the current policy: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`
- If SSL policy change breaks clients, revert to the previous policy
- If certificate replacement causes issues, re-attach the previous certificate
- If additional certificates were added via SNI, remove them if they cause conflicts
- Verify TLS connections work correctly after any changes
