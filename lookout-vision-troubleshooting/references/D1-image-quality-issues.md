---
title: "D1 — Image Quality Issues"
description: "Diagnose image quality problems affecting model training and inference accuracy"
status: active
severity: MEDIUM
triggers: ["poor image quality", "blurry images", "inconsistent lighting", "image resolution", "camera setup"]
owner: devops-agent
objective: "Improve image quality for better model training and inference accuracy"
context: "Image quality is the single most important factor in model accuracy. Consistent lighting, camera angle, distance, and resolution are critical. Variations introduce noise that degrades performance."
---

## Phase 1 — Triage

MUST:
- Review sample training images for quality consistency
- Check image resolution — minimum 64x64, recommended 512x512+
- Verify image format is PNG or JPEG

SHOULD:
- Assess lighting consistency across images
- Check camera angle and distance consistency
- Verify background consistency (clean, uniform background preferred)

MAY:
- Compare image quality between normal and anomaly classes
- Check if image capture conditions changed between training and inference

## Phase 2 — Remediate

MUST:
- Standardize camera position with a fixed mount
- Ensure consistent lighting (avoid shadows, reflections, variable ambient light)
- Use consistent image resolution across all captures

SHOULD:
- Use a uniform background to reduce noise
- Capture images at the same distance from the subject
- Ensure the subject fills a consistent portion of the frame

MAY:
- Set up automated image capture with a fixed camera system
- Implement image quality validation before adding to dataset
- Use image preprocessing to normalize brightness and contrast

## Common Issues

- symptoms: "Model accuracy is poor despite sufficient training data"
  diagnosis: "Inconsistent image quality — varying lighting, angles, or distances."
  resolution: "Standardize image capture setup. Recapture training images with consistent conditions. Retrain model."

- symptoms: "Model works well on some images but poorly on others"
  diagnosis: "Inference images have different characteristics than training images."
  resolution: "Match inference capture conditions to training conditions. Add diverse examples to training data."

## Safety Ratings

```
safety_ratings:
  - "Image quality review: GREEN — visual inspection only"
  - "Camera setup changes: YELLOW — affects all future image captures"
  - "Dataset recapture: YELLOW — requires time and resources"
  - "Model retraining: YELLOW — incurs compute charges"
```

## Escalation Conditions

- Image quality cannot be improved due to physical constraints (space, equipment)
- Camera system requires capital investment for upgrade
- Production line cannot be stopped for camera setup changes
- Image quality requirements conflict with production speed

## Data Sensitivity

- Training images: HIGH — may contain proprietary product images
- Camera setup details: LOW — equipment configuration
- Image quality metrics: LOW — technical measurements
- Production line images: HIGH — may reveal manufacturing processes

## Prohibited Actions

- NEVER use images from different camera setups in the same training dataset without normalization
- NEVER assume image quality is acceptable without visual inspection
- NEVER share production line images outside the organization
- NEVER modify images in ways that alter defect characteristics (over-processing)

## Phase 3 — Rollback

1. If camera setup was changed and quality worsened, restore previous setup
2. If images were preprocessed incorrectly, restore originals from backup
3. If model was retrained with poor quality images, revert to previous model version
4. Document image quality standards for the capture team
5. Verify image quality meets standards before next training cycle

## Output Format

```yaml
root_cause: "image_quality — <specific_cause>"
evidence:
  - type: quality_issue
    content: "<resolution, lighting, consistency details>"
severity: MEDIUM
mitigation:
  immediate: "Standardize capture conditions"
  long_term: "Implement automated image quality validation"
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
