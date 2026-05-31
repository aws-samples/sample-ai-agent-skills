---
title: "C1 — Model Compilation Failures"
description: "Diagnose ML model compilation issues for Panorama"
status: active
severity: HIGH
triggers:
  - "model compilation failed"
  - "SageMaker Neo error"
  - "unsupported operator"
owner: devops-agent
objective: "Resolve model compilation failures for Panorama deployment"
context: "Models must be compiled via SageMaker Neo targeting the Panorama appliance GPU. Supported frameworks: TensorFlow, PyTorch, MXNet, ONNX."
---

## Phase 1 — Triage

MUST:
- Check compilation job status: `aws sagemaker describe-compilation-job --compilation-job-name <name>`
- Verify target device is set to panorama (jetson_xavier)
- Check input model format matches declared framework
- Review compilation job logs in CloudWatch

SHOULD:
- Verify model input shape is correctly specified
- Check for unsupported operators in the model
- Verify S3 paths for input model and output artifacts

MAY:
- Test model locally before compilation
- Check SageMaker Neo supported operator list

## Phase 2 — Remediate

MUST:
- Fix unsupported operators by replacing with supported alternatives
- Correct input shape specification
- Ensure S3 permissions for model artifacts

SHOULD:
- Use latest framework version for better operator support
- Simplify model architecture if compilation fails repeatedly

## Common Issues

- symptoms: "Compilation failed: unsupported operator"
  diagnosis: "Model uses operators not supported by Neo for Panorama target."
  resolution: "Replace unsupported operators. Check Neo supported operators documentation."

- symptoms: "Compilation timeout"
  diagnosis: "Model too large or complex for compilation within time limit."
  resolution: "Simplify model or increase compilation timeout. Consider model optimization."

## Output Format

```yaml
root_cause: "model_compilation — <specific_cause>"
evidence:
  - type: compilation_status
    content: "<compilation job status and errors>"
severity: HIGH
mitigation:
  immediate: "Fix model and recompile"
  long_term: "Validate model compatibility before compilation"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — model artifacts may be proprietary
- Service Disruption Risk: LOW — compilation failures do not affect running models
- Compliance Impact: LOW — compilation is a development activity

## Escalation Conditions
- Model compilation fails with unsupported operator errors
- Compilation produces incorrect model output
- Compilation exceeds resource or time limits
- Model framework version incompatibility
- Compiled model performance significantly worse than expected

## Data Sensitivity
- Model architecture and weights — CONFIDENTIAL
- Training data references — CONFIDENTIAL
- Compilation configurations — INTERNAL
- Model performance metrics — INTERNAL
- Never expose proprietary model details

## Prohibited Actions
- Do NOT compile models with untrusted frameworks
- Do NOT skip model validation after compilation
- Do NOT deploy unvalidated compiled models to production
- Do NOT share model artifacts with unauthorized parties
- Do NOT ignore compilation warnings
- Do NOT use deprecated model frameworks without migration plan

## Phase 3 — Rollback
- Revert to previous compiled model version
- Restore compilation configurations
- Re-compile with corrected parameters
- Verify model accuracy after re-compilation
- Document compilation issues and framework requirements
- Update model compilation procedures

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
