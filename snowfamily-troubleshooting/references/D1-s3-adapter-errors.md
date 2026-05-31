---
title: "D1 — S3 Adapter Errors"
description: "Diagnose Snowball Edge S3 adapter issues"
status: active
severity: HIGH
triggers: ["S3 adapter error", "S3 on Snow failed", "bucket error on device", "S3 endpoint error"]
owner: devops-agent
objective: "Resolve S3 adapter issues including service status, API compatibility, and endpoint configuration"
context: "S3 adapter provides S3-compatible API on Snow devices. Supports a subset of S3 operations. Endpoint is https://<device-ip>:8443."
---

## Phase 1 — Triage

MUST:
- Check S3 adapter service: `snowballEdge describe-service --service-id s3 --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Test S3 endpoint: `aws s3 ls --endpoint https://<device-ip>:8443 --profile snowball`
- Verify S3 adapter is running and accessible

SHOULD:
- Check available storage on device
- Verify AWS CLI profile is configured with device credentials

MAY:
- Test specific S3 operations (PutObject, GetObject, ListObjects)
- Check device certificate for HTTPS connectivity

## Phase 2 — Remediate

MUST:
- Start S3 adapter if not running: `snowballEdge start-service --service-id s3 --virtual-network-interface-arns <vni-arn>`
- Configure AWS CLI profile with device access key and secret key
- Use correct endpoint URL: `https://<device-ip>:8443`

SHOULD:
- Create buckets before attempting to write objects
- Use --no-verify-ssl if certificate issues occur (development only)

MAY:
- Configure S3 adapter with a virtual network interface for dedicated networking

## Safety Ratings

```
safety_ratings:
  - "describe-service: GREEN — read-only service status"
  - "s3 ls (device): GREEN — read-only bucket listing"
  - "start-service: YELLOW — starts S3 adapter on device"
  - "s3 cp to device: YELLOW — writes data to device storage"
```

## Escalation Conditions

- S3 adapter service crashes repeatedly
- S3 API operations return unexpected errors not in documentation
- Device storage is corrupted or inaccessible
- S3 adapter performance is significantly below expected throughput

## Data Sensitivity

- `snowballEdge describe-service`: LOW — reveals service configuration
- `aws s3 ls (device)`: MEDIUM — reveals bucket and object listing
- S3 adapter access credentials: HIGH — provide full access to device storage

## Prohibited Actions

- NEVER use unsupported S3 API operations (versioning, lifecycle, bucket policies)
- NEVER expose S3 adapter endpoint to untrusted networks
- NEVER share device S3 credentials outside the authorized team
- NEVER assume S3 adapter behavior matches Amazon S3 exactly

## Phase 3 — Rollback

1. If S3 adapter was restarted with wrong configuration, stop and reconfigure
2. If incorrect data was uploaded, delete it from the device
3. Verify existing data integrity after S3 adapter restart
4. Restore AWS CLI profile configuration if modified
5. Document S3 adapter endpoint and credentials for the team

## Common Issues

- symptoms: "aws s3 ls returns 'Could not connect to the endpoint URL'"
  diagnosis: "S3 adapter service is not running or endpoint URL is incorrect."
  resolution: "Start S3 adapter service. Use https://<device-ip>:8443 as endpoint. Verify network connectivity."

## Output Format

```yaml
root_cause: "s3_adapter — <specific_cause>"
evidence:
  - type: s3_adapter_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 adapter"
  long_term: "Document S3 adapter limitations"
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
