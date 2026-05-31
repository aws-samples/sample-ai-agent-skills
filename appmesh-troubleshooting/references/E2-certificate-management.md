---
title: "E2 — App Mesh Certificate Management Issues"
description: "Diagnose certificate provisioning, rotation, and expiration problems"
status: active
severity: MEDIUM
triggers:
  - "certificate expired"
  - "certificate rotation"
  - "ACM-PCA"
  - "certificate provisioning"
  - "CA error"
owner: devops-agent
objective: "Identify and resolve certificate management issues affecting App Mesh TLS"
context: "App Mesh uses ACM-PCA for certificate management. Certificates are automatically rotated by Envoy. Issues arise from CA configuration, certificate expiration, or rotation failures."
---

## Phase 1 — Triage

MUST:
- Check ACM-PCA CA status: `aws acm-pca describe-certificate-authority --certificate-authority-arn <ca-arn>`
- List certificates: `aws acm-pca list-certificates --certificate-authority-arn <ca-arn>`
- Check CA certificate expiration: `aws acm-pca get-certificate-authority-certificate --certificate-authority-arn <ca-arn>`
- Verify Envoy certificate status: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/certs`

SHOULD:
- Check certificate validity period
- Verify CA chain is complete and valid
- Check Envoy SDS (Secret Discovery Service) status

MAY:
- Check ACM-PCA audit report for certificate issuance history
- Verify CA CRL (Certificate Revocation List) configuration

## Phase 2 — Remediate

MUST:
- Renew or replace expired CA certificates
- Ensure ACM-PCA CA is in ACTIVE status
- Restart pods if certificates are stale after CA renewal

SHOULD:
- Set up CA certificate expiration monitoring
- Configure appropriate certificate validity periods
- Test certificate rotation in staging

MAY:
- Implement CA certificate rotation procedures
- Set up CloudWatch alarms for certificate expiration

## Common Issues

- symptoms: "TLS errors after CA certificate expiration"
  diagnosis: "ACM-PCA CA certificate has expired."
  resolution: "Renew the CA certificate and restart affected pods."

- symptoms: "New pods fail TLS but existing pods work"
  diagnosis: "CA is in a degraded state or certificate issuance is failing."
  resolution: "Check CA status and certificate issuance logs."

- symptoms: "Certificate rotation not happening"
  diagnosis: "Envoy SDS not refreshing certificates from App Mesh."
  resolution: "Check Envoy SDS connection. Restart pod to force certificate refresh."

## Output Format

```yaml
root_cause: "certificate_management_issue — <specific_cause>"
evidence:
  - type: ca_status
    content: "<ACM-PCA CA status and expiration>"
  - type: certificates
    content: "<certificate details and validity>"
  - type: envoy_certs
    content: "<Envoy certificate status>"
severity: MEDIUM
mitigation:
  immediate: "Renew certificates or fix CA configuration"
  long_term: "Implement certificate lifecycle monitoring and alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Renew or replace expired CA certificates | RED | CA change affects all services |
| Ensure ACM-PCA CA is ACTIVE | GREEN | Status verification |
| Restart pods for certificate refresh | YELLOW | Causes brief service disruption |
| Set up CA certificate expiration monitoring | GREEN | Monitoring setup |
| Configure certificate validity periods | GREEN | Policy configuration |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **CA certificates**: Root and intermediate CA details — CONFIDENTIAL
- **Certificate validity**: Expiration dates and rotation status — INTERNAL
- **Envoy SDS status**: Secret discovery service state — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER let CA certificates expire without a rotation plan
- NEVER revoke CA certificates without understanding the blast radius

## Phase 3 — Rollback

1. If CA was renewed, verify all services can validate the new CA
2. If pods were restarted, verify they received valid certificates
3. Restore previous CA certificate if the new one causes issues
4. Check Envoy certificate status: `kubectl exec <pod> -c envoy -- curl -s http://localhost:9901/certs`
5. Monitor TLS handshake success rates across all services

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
