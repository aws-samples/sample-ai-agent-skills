---
title: "E1 — App Mesh mTLS Issues"
description: "Diagnose mutual TLS configuration and handshake failures"
status: active
severity: HIGH
triggers:
  - "mTLS"
  - "mutual TLS"
  - "TLS handshake"
  - "certificate error"
  - "SSL error"
owner: devops-agent
objective: "Identify and resolve mTLS configuration issues in App Mesh"
context: "App Mesh mTLS uses ACM Private CA certificates for mutual authentication between services. Both client and server must present valid certificates. Misconfiguration causes TLS handshake failures and connection refused errors."
---

## Phase 1 — Triage

MUST:
- Check virtual node TLS config: `aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name> --query 'virtualNode.spec.listeners[0].tls'`
- Check backend TLS config: `aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name> --query 'virtualNode.spec.backendDefaults'`
- Verify ACM-PCA CA is active: `aws acm-pca list-certificate-authorities --query 'CertificateAuthorities[*].{Arn:Arn,Status:Status}'`
- Check Envoy TLS stats: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/stats | grep ssl`
- Check Envoy logs for TLS errors: `kubectl logs <pod-name> -c envoy | grep -i "tls\|ssl\|certificate"`

SHOULD:
- Verify certificate chain is valid
- Check certificate expiration dates
- Verify Subject Alternative Names (SANs) match expected values

MAY:
- Check Envoy config dump for TLS context configuration
- Verify certificate rotation is working

## Phase 2 — Remediate

MUST:
- Use ACM-PCA certificates (NOT public ACM certificates) for mTLS
- Configure TLS on virtual node listeners with correct certificate source
- Configure client TLS validation on backend defaults
- Ensure ACM-PCA CA is in ACTIVE status

SHOULD:
- Set up SAN validation for additional security
- Configure certificate auto-rotation
- Test mTLS in a staging environment before production

MAY:
- Enable TLS inspection logging for debugging
- Implement certificate monitoring and alerting

## Common Issues

- symptoms: "TLS handshake failure between services"
  diagnosis: "Client and server certificates not from the same CA or CA not trusted."
  resolution: "Ensure both services use certificates from the same ACM-PCA CA."

- symptoms: "Connection refused after enabling mTLS"
  diagnosis: "One side has mTLS enabled but the other doesn't have client certificate configured."
  resolution: "Configure both client and server TLS settings on all participating virtual nodes."

- symptoms: "Certificate validation fails with SAN mismatch"
  diagnosis: "Certificate SAN doesn't match the virtual service name."
  resolution: "Issue certificates with SANs matching the virtual service DNS names."

## Output Format

```yaml
root_cause: "mtls_issue — <specific_cause>"
evidence:
  - type: tls_config
    content: "<listener and backend TLS settings>"
  - type: certificate
    content: "<CA status and certificate details>"
  - type: envoy_tls_stats
    content: "<TLS handshake statistics>"
severity: HIGH
mitigation:
  immediate: "Fix certificate configuration or CA trust"
  long_term: "Implement certificate monitoring and automated rotation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use ACM-PCA certificates for mTLS | YELLOW | Certificate infrastructure change |
| Configure TLS on virtual node listeners | YELLOW | Encryption configuration change |
| Configure client TLS validation | YELLOW | Changes authentication requirements |
| Ensure ACM-PCA CA is ACTIVE | GREEN | Status verification |
| Set up SAN validation | GREEN | Security hardening |
| Test mTLS in staging | GREEN | Non-production testing |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **TLS certificates**: ACM-PCA certificates and CA details — CONFIDENTIAL
- **Certificate SANs**: Service identity information — INTERNAL
- **Envoy TLS stats**: Handshake success/failure metrics — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER disable TLS validation to work around certificate errors
- NEVER use self-signed certificates in production mesh

## Phase 3 — Rollback

1. Revert TLS configuration on affected virtual nodes to previous state
2. If mTLS was newly enabled, disable it on both client and server sides simultaneously
3. Restart affected pods to pick up reverted TLS configuration
4. Verify service-to-service communication is restored
5. Check Envoy TLS stats to confirm handshakes are succeeding

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
