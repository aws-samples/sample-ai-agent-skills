---
title: "B1 — Decoder Manifest Issues"
description: "Diagnose decoder manifest creation, signal mapping, and activation failures"
status: active
severity: HIGH
triggers: ["decoder manifest error", "signal decoding failed", "CAN decoder issue", "OBD decoder error", "manifest validation failed"]
owner: devops-agent
objective: "Resolve decoder manifest configuration issues for proper signal decoding"
context: "Decoder manifests define how vehicle network signals (CAN, OBD-II) are translated to cloud signals. Each model manifest requires at least one decoder manifest. Must be ACTIVE before use."
---

## Phase 1 — Triage

MUST:
- List decoder manifests: `aws iotfleetwise list-decoder-manifests --query 'summaries[*].{Name:name,Status:status,ModelManifestArn:modelManifestArn}'`
- Describe the failing manifest: `aws iotfleetwise get-decoder-manifest --name <name>`
- List decoder signals: `aws iotfleetwise list-decoder-manifest-signals --name <name>`

SHOULD:
- Verify associated model manifest is ACTIVE
- Check decoder definitions match expected CAN/OBD-II signal specifications
- Compare decoder signal list with model manifest signal list for completeness

MAY:
- Review CloudTrail for decoder manifest modification events
- Validate CAN database (DBC) file if used as source for decoder definitions
- Check network interface definitions in the decoder manifest

## Phase 2 — Remediate

MUST:
- Ensure decoder manifest references a valid, ACTIVE model manifest
- Define decoders for all signals in the associated model manifest
- Activate decoder manifest: `aws iotfleetwise update-decoder-manifest --name <name> --status ACTIVE`

SHOULD:
- Verify CAN signal definitions: message ID, start bit, length, byte order, scaling, offset
- Verify OBD-II signal definitions: PID, service mode, scaling
- Ensure network interface type matches vehicle hardware (CAN_INTERFACE or OBD_INTERFACE)

MAY:
- Import decoder definitions from DBC file for CAN signals
- Create separate decoder manifests for different vehicle hardware variants
- Document decoder specifications for the vehicle engineering team

## Safety Ratings

```
safety_ratings:
  - "list-decoder-manifests: GREEN — read-only enumeration"
  - "get-decoder-manifest: GREEN — read-only inspection"
  - "update-decoder-manifest --status ACTIVE: YELLOW — activates decoder, enables data collection"
  - "delete-decoder-manifest: RED — removes decoder, breaks dependent vehicles"
```

## Escalation Conditions

- Decoder definitions produce incorrect signal values indicating specification error
- CAN message IDs conflict between different vehicle models
- Decoder manifest changes needed for production vehicles during active campaigns
- Vehicle hardware variant requires new network interface type not yet supported

## Data Sensitivity

- `aws iotfleetwise get-decoder-manifest`: LOW — returns decoder configuration
- `aws iotfleetwise list-decoder-manifest-signals`: LOW — returns signal mapping definitions
- CAN/OBD-II specifications: MEDIUM — may be proprietary vehicle data

## Prohibited Actions

- NEVER delete a decoder manifest associated with active vehicles
- NEVER activate a decoder manifest without verifying signal mappings are correct
- NEVER modify decoder definitions for production vehicles without testing on a staging vehicle first
- NEVER assume CAN signal byte order — verify big-endian vs little-endian for each signal

## Phase 3 — Rollback

1. If decoder was incorrectly activated, create a corrected decoder manifest
2. Update vehicles to use the corrected decoder manifest
3. If signals were incorrectly mapped, update decoder signal definitions
4. Verify data collection produces correct values after decoder changes
5. Document the decoder specification corrections for future reference

## Common Issues

- symptoms: "Decoder manifest activation fails with 'signals not covered' error"
  diagnosis: "Decoder manifest does not define decoders for all signals in the associated model manifest."
  resolution: "Add missing signal decoders. List model manifest nodes and ensure each has a corresponding decoder definition."

- symptoms: "Data collected shows incorrect values (wrong scaling or offset)"
  diagnosis: "CAN signal decoder has incorrect scaling factor, offset, bit position, or byte order."
  resolution: "Verify decoder parameters against the vehicle's CAN database (DBC file). Fix scaling, offset, startBit, and length values."

## Output Format

```yaml
root_cause: "decoder_manifest — <specific_cause>"
evidence:
  - type: decoder_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix decoder signal mappings"
  long_term: "Implement decoder validation against DBC specifications"
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
