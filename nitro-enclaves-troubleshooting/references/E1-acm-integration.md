---
title: "E1 — ACM Integration"
description: "Diagnose ACM for Nitro Enclaves certificate delivery and TLS configuration issues"
status: active
severity: HIGH
triggers: ["ACM certificate not delivered", "acm-for-nitro-enclaves error", "TLS certificate missing in enclave", "ACM helper failed"]
owner: devops-agent
objective: "Resolve ACM integration issues for TLS certificate delivery to enclaves"
context: "ACM for Nitro Enclaves delivers TLS certificates to enclaves without exposing private keys to the parent instance. Requires the acm-for-nitro-enclaves helper service on the parent, IAM role with ACM permissions, and enclave-side certificate handling."
---

## Phase 1 — Triage

MUST:
- Check ACM helper service status: `systemctl status acm-for-nitro-enclaves.service`
- Verify ACM certificate exists: `aws acm describe-certificate --certificate-arn <arn> --query '{Status:Certificate.Status,DomainName:Certificate.DomainName}'`
- Check enclave is running: `nitro-cli describe-enclaves`

SHOULD:
- Review ACM helper configuration: `cat /etc/nitro_enclaves/acm.yaml`
- Verify IAM role has acm:ExportCertificate and acm:DescribeCertificate permissions
- Check enclave console for certificate-related errors: `nitro-cli console --enclave-id <id>`

MAY:
- Review ACM helper logs: `journalctl -u acm-for-nitro-enclaves.service`
- Verify certificate is issued (not pending validation)
- Check if certificate is associated with the correct enclave configuration

## Phase 2 — Remediate

MUST:
- Start ACM helper service if not running: `sudo systemctl start acm-for-nitro-enclaves.service`
- Ensure ACM certificate is in ISSUED status
- Configure acm.yaml with correct certificate ARN and enclave CID

SHOULD:
- Verify IAM instance profile has acm:ExportCertificate permission
- Ensure ACM helper is configured to deliver to the correct vsock port
- Check certificate renewal status for expiring certificates

MAY:
- Enable ACM helper debug logging for detailed diagnostics
- Set up certificate renewal monitoring
- Test with a self-signed certificate to isolate ACM-specific issues

## Safety Ratings

```
safety_ratings:
  - "systemctl status: GREEN — read-only service check"
  - "acm describe-certificate: GREEN — read-only certificate info"
  - "describe-enclaves: GREEN — read-only enclave info"
  - "systemctl start acm-helper: YELLOW — starts certificate delivery service"
  - "acm export-certificate: YELLOW — exports certificate material"
```

## Escalation Conditions

- ACM certificate is in PENDING_VALIDATION status and cannot be issued
- ACM helper service crashes repeatedly
- Certificate private key exposure suspected (security incident)
- Cross-account ACM certificate access required

## Data Sensitivity

- `aws acm describe-certificate`: LOW — returns certificate metadata
- ACM helper configuration: MEDIUM — contains certificate ARN and delivery settings
- Certificate material: CRITICAL — private keys must never be exposed outside enclave
- ACM helper logs: MEDIUM — may contain certificate delivery details

## Prohibited Actions

- NEVER export ACM certificate private keys to the parent instance filesystem
- NEVER log certificate material in ACM helper or application logs
- NEVER use expired or self-signed certificates in production enclaves
- NEVER disable ACM helper service while enclave requires TLS certificates

## Phase 3 — Rollback

1. If ACM helper was misconfigured, restore previous acm.yaml and restart service
2. If certificate was deleted, request a new certificate from ACM
3. If enclave cannot receive certificate, terminate and restart enclave
4. Verify TLS connections work after certificate delivery is restored
5. Monitor certificate expiry and renewal status

## Common Issues

- symptoms: "ACM helper service fails to start with permission error"
  diagnosis: "IAM instance profile does not have acm:ExportCertificate permission for the certificate ARN."
  resolution: "Add acm:ExportCertificate and acm:DescribeCertificate permissions to the instance profile IAM role."

- symptoms: "Certificate delivered but enclave application cannot use it for TLS"
  diagnosis: "Enclave application is not reading the certificate from the expected vsock port, or certificate format is incompatible."
  resolution: "Verify enclave application reads certificate from the vsock port configured in acm.yaml. Check certificate format compatibility."

## Output Format

```yaml
root_cause: "acm_integration — <specific_cause>"
evidence:
  - type: acm_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix ACM helper configuration"
  long_term: "Implement certificate monitoring and auto-renewal"
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
