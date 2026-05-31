---
title: "H2 — IMDS (Instance Metadata Service) Issues"
description: "Diagnose IMDS accessibility and credential delivery failures"
status: active
severity: HIGH
triggers:
  - "169.254.169.254.*timeout"
  - "Unable to locate credentials"
  - "EC2MetadataError"
  - "metadata.*unreachable"
owner: devops-agent
objective: "Restore IMDS access for credential delivery and metadata retrieval"
context: "IMDS provides instance metadata and temporary IAM credentials at 169.254.169.254. IMDSv2 uses session tokens (PUT then GET). Hop limit controls how many network hops can reach IMDS (1 = host only, 2 = containers). IMDS can be disabled entirely."
---

## Phase 1 — Triage

MUST:
- Check IMDS configuration: `aws ec2 describe-instances --instance-ids <id>` → MetadataOptions
  - HttpEndpoint: enabled/disabled
  - HttpTokens: required (IMDSv2 only) / optional (v1 and v2)
  - HttpPutResponseHopLimit: 1 or 2
- If HttpEndpoint=disabled: IMDS is completely off — that's the root cause
- If HttpPutResponseHopLimit=1 and running containers: containers can't reach IMDS

SHOULD:
- Check if iptables/firewall rules block 169.254.169.254
- Check if a proxy is intercepting metadata requests
- Verify the SDK/CLI version supports IMDSv2 (if HttpTokens=required)

MAY:
- Test IMDS from the instance via SSM: `curl -s http://169.254.169.254/latest/meta-data/`
- For IMDSv2: `TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600") && curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/`

## Common Issues

- symptoms: "Containers cannot get IAM credentials, host can"
  diagnosis: "HttpPutResponseHopLimit is 1. Containers need hop limit 2 (extra network hop)."
  resolution: "`aws ec2 modify-instance-metadata-options --instance-id <id> --http-put-response-hop-limit 2`"

- symptoms: "All IMDS requests fail (host and containers)"
  diagnosis: "IMDS disabled (HttpEndpoint=disabled) or network/firewall blocking 169.254.169.254."
  resolution: "Enable IMDS: `aws ec2 modify-instance-metadata-options --instance-id <id> --http-endpoint enabled`"

- symptoms: "IMDSv1 works but IMDSv2 fails"
  diagnosis: "SDK/CLI too old to support IMDSv2 token flow, or proxy stripping PUT headers."
  resolution: "Update SDK/CLI. If proxy issue, configure proxy to pass through IMDS headers."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances MetadataOptions: GREEN — read-only"
  - "curl IMDS endpoint via SSM: GREEN — read-only metadata retrieval"
  - "Enable IMDS (http-endpoint enabled): YELLOW — enables metadata access, recoverable"
  - "Increase hop limit to 2: YELLOW — extends IMDS reach to containers, recoverable"
  - "Switch to IMDSv2 required: YELLOW — may break old SDKs/scripts, recoverable by switching back"
  - "Disable IMDS entirely: RED — breaks all credential delivery and metadata access"
```

## Escalation Conditions
- IMDS is disabled on a production instance and applications depend on instance metadata
- Switching to IMDSv2-required would break legacy applications that only support IMDSv1
- IMDS issue is caused by a network/firewall rule that affects multiple instances
- Container workloads cannot reach IMDS and require hop limit change across a fleet
- IMDS configuration is managed by a security policy and cannot be changed without approval

## Data Sensitivity
- HIGH: IMDS responses (contain IAM credentials, instance identity, network configuration, user-data)
- HIGH: describe-instances MetadataOptions (reveals IMDS security configuration)
- MEDIUM: curl IMDS output via SSM (reveals metadata categories available)

## Prohibited Actions
- NEVER suggest disabling IMDS to troubleshoot credential issues
- NEVER suggest using IMDSv1 (http-tokens optional) as a permanent fix when IMDSv2 is required by policy
- NEVER suggest increasing hop limit beyond 2 without understanding the network topology
- NEVER suggest hardcoding credentials as a workaround for IMDS issues

## Phase 3 — Rollback
- If IMDS was enabled: disable with `modify-instance-metadata-options --http-endpoint disabled`
- If hop limit was changed: revert with `modify-instance-metadata-options --http-put-response-hop-limit <original>`
- If IMDSv2 was enforced: revert with `modify-instance-metadata-options --http-tokens optional`
- If firewall rules were modified: revert iptables/firewall rules via SSM

## Output Format

```yaml
root_cause: "<disabled|hop_limit|imdsv2_incompatible|firewall_block|proxy> — <detail>"
evidence:
  - type: metadata_options
    content: "<MetadataOptions from describe-instances>"
severity: HIGH
mitigation:
  immediate: "Enable IMDS, increase hop limit, or update SDK"
  long_term: "Standardize IMDSv2 with hop limit 2 in launch templates"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
