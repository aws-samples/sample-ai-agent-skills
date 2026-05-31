---
title: "B2 — Vsock Communication"
description: "Diagnose vsock communication failures between parent instance and enclave"
status: active
severity: HIGH
triggers: ["vsock error", "cannot communicate with enclave", "vsock connection refused", "parent-enclave communication failed"]
owner: devops-agent
objective: "Resolve vsock communication issues between parent and enclave"
context: "Vsock is the ONLY communication channel between parent and enclave. It uses a CID (Context Identifier) and port number. The enclave CID is assigned at creation. Parent CID is always 3. Communication failures break all data exchange."
---

## Phase 1 — Triage

MUST:
- Get enclave CID: `nitro-cli describe-enclaves --query '[*].{EnclaveID:EnclaveID,EnclaveCID:EnclaveCID,State:State}'`
- Verify enclave is running: check State is RUNNING
- Check if vsock proxy or application is listening on expected port

SHOULD:
- Review enclave console for vsock-related errors: `nitro-cli console --enclave-id <id>`
- Verify parent-side vsock client/server is running and using correct CID and port
- Check vsock kernel module: `lsmod | grep vsock`

MAY:
- Test vsock connectivity with a simple echo server/client
- Review application logs on parent for connection errors
- Check if SELinux or security policies block vsock access

## Phase 2 — Remediate

MUST:
- Ensure enclave is in RUNNING state before attempting vsock communication
- Use correct CID: enclave CID from describe-enclaves, parent CID is always 3
- Verify both parent and enclave applications use the same port number

SHOULD:
- Ensure vsock listener starts before client attempts connection
- Implement connection retry logic in the parent application
- Verify vsock buffer sizes are appropriate for data volume

MAY:
- Implement heartbeat mechanism over vsock for connection health monitoring
- Use the Nitro Enclaves SDK for simplified vsock communication
- Profile vsock throughput for performance-sensitive applications

## Safety Ratings

```
safety_ratings:
  - "describe-enclaves: GREEN — read-only enclave info"
  - "console: GREEN — read-only console output"
  - "lsmod: GREEN — read-only kernel module check"
  - "vsock test tools: GREEN — diagnostic communication test"
```

## Escalation Conditions

- Vsock communication fails consistently indicating kernel or driver issue
- High-throughput vsock communication causes performance degradation
- Vsock connection drops during critical data processing
- Security policy changes block vsock access requiring policy review

## Data Sensitivity

- `nitro-cli describe-enclaves`: LOW — returns CID and state
- `nitro-cli console`: MEDIUM — may show data processing details
- Vsock traffic: HIGH — carries sensitive data between parent and enclave
- Application logs: MEDIUM — may contain connection details

## Prohibited Actions

- NEVER attempt network-based communication with the enclave — only vsock works
- NEVER hardcode CID values — always get from describe-enclaves
- NEVER send unencrypted sensitive data over vsock without understanding the threat model
- NEVER ignore vsock connection failures — they indicate the enclave cannot function

## Phase 3 — Rollback

1. If vsock communication is broken, terminate and restart the enclave
2. If parent application was modified, restore previous version
3. If kernel module was unloaded, reload: `sudo modprobe vsock`
4. Verify vsock communication works with a simple test before deploying application
5. Monitor vsock stability for 15 minutes after restoration

## Common Issues

- symptoms: "Connection refused when parent tries to connect to enclave vsock"
  diagnosis: "Enclave application is not listening on the expected vsock port, or enclave is not running."
  resolution: "Verify enclave is RUNNING. Check enclave application starts vsock listener on the expected port. Verify CID is correct."

- symptoms: "Vsock connection established but data transfer hangs"
  diagnosis: "Buffer sizes too small for data volume, or application protocol mismatch between parent and enclave."
  resolution: "Increase vsock buffer sizes. Verify parent and enclave use compatible message framing protocol."

## Output Format

```yaml
root_cause: "vsock_communication — <specific_cause>"
evidence:
  - type: vsock_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix vsock communication"
  long_term: "Implement vsock health monitoring and reconnection logic"
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
