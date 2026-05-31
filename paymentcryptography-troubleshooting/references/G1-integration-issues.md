---
title: "G1 — Integration Issues"
description: "Diagnose Payment Cryptography integration issues with payment systems and applications"
status: active
severity: MEDIUM
triggers: ["integration error", "SDK error", "API Gateway", "payment processor integration"]
owner: devops-agent
objective: "Resolve integration issues between Payment Cryptography and payment systems"
context: "Payment Cryptography integrates with payment processing applications via SDK/API. Control plane and data plane have separate endpoints."
---

## Phase 1 — Triage

MUST:
- Verify API endpoint connectivity: `aws payment-cryptography list-keys --max-results 1`
- Verify data plane connectivity: `aws payment-cryptography-data --region <region> help`
- Check IAM role trust policies for service integrations

SHOULD:
- Verify SDK version supports Payment Cryptography APIs
- Check VPC endpoint configuration if using private access

MAY:
- Test API calls with verbose logging: `aws payment-cryptography list-keys --debug`
- Check CloudWatch Logs for application-level integration errors

## Phase 2 — Remediate

MUST:
- Update SDK to version supporting Payment Cryptography
- Fix IAM role trust policies for cross-service access
- Configure correct regional endpoints for both control and data planes

SHOULD:
- Implement health checks for API connectivity
- Use VPC endpoints for private network access

MAY:
- Set up CloudWatch dashboards for integration monitoring
- Implement circuit breaker patterns for payment operations

## Safety Ratings

```
safety_ratings:
  - "list-keys: GREEN — read-only connectivity test"
  - "VPC endpoint creation: YELLOW — creates network infrastructure"
  - "IAM role trust policy modification: YELLOW — changes service access"
  - "SDK update: YELLOW — changes application dependencies"
```

## Escalation Conditions

- Payment Cryptography API unreachable from VPC
- SDK compatibility issues require application code changes
- Integration with external payment processor requires custom development
- Latency exceeds payment processing SLA requirements

## Data Sensitivity

- API responses: LOW to HIGH — depends on operation (key metadata vs payment data)
- Application logs: HIGH — may contain payment transaction details
- VPC endpoint configuration: LOW — network metadata
- SDK configuration: LOW — library settings

## Prohibited Actions

- NEVER expose Payment Cryptography endpoints publicly without authentication
- NEVER log payment data during integration debugging
- NEVER use public endpoints when VPC endpoints are required by compliance
- NEVER bypass TLS for API connectivity troubleshooting

## Phase 3 — Rollback

1. If VPC endpoints were created: delete if not needed
2. If IAM trust policies were modified: revert to previous configuration
3. If SDK was updated: rollback to previous version if issues arise
4. Verify all payment processing integrations are functioning
5. Document integration changes for operations team

## Common Issues

- symptoms: "SDK throws UnknownServiceError for payment-cryptography-data"
  diagnosis: "SDK version does not support Payment Cryptography data plane."
  resolution: "Update AWS SDK to latest version. Payment Cryptography requires recent SDK versions."

- symptoms: "High latency on data plane operations"
  diagnosis: "Network path to data plane endpoint is suboptimal or VPC endpoint not configured."
  resolution: "Configure VPC endpoint for payment-cryptography-data. Check network routing and security groups."

## Output Format

```yaml
root_cause: "integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix integration issue"
  long_term: "Implement integration monitoring"
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
