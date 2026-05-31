---
title: "A2 — Enclave Image Build"
description: "Diagnose enclave image (EIF) build failures from Docker images"
status: active
severity: MEDIUM
triggers: ["EIF build failed", "build-enclave error", "Docker image to EIF failed", "enclave image invalid"]
owner: devops-agent
objective: "Resolve enclave image build failures and produce valid EIF files"
context: "Enclave images (EIF) are built from Docker images using nitro-cli build-enclave. The Docker image must be self-contained since enclaves have no network access. Build failures are typically Docker or dependency related."
---

## Phase 1 — Triage

MUST:
- Verify Docker is running: `docker info`
- Check Docker image exists: `docker images <image-name>`
- Attempt build with verbose output: `nitro-cli build-enclave --docker-uri <image> --output-file test.eif`

SHOULD:
- Inspect Docker image for size and layers: `docker inspect <image>`
- Verify Docker image runs correctly outside enclave: `docker run --rm <image>`
- Check available disk space for EIF build: `df -h`

MAY:
- Review Dockerfile for network-dependent operations in ENTRYPOINT/CMD
- Check nitro-cli version compatibility: `nitro-cli --version`
- Verify Docker image architecture matches instance architecture (x86_64 vs arm64)

## Phase 2 — Remediate

MUST:
- Ensure Docker daemon is running and accessible
- Fix Docker image issues (missing dependencies, incorrect entrypoint)
- Ensure Docker image is self-contained — no runtime network dependencies

SHOULD:
- Minimize Docker image size to reduce EIF size and memory requirements
- Use multi-stage Docker builds to exclude build tools from final image
- Verify ENTRYPOINT or CMD is set in the Dockerfile

MAY:
- Build with signing for attestation: `nitro-cli build-enclave --docker-uri <image> --output-file <output.eif> --signing-certificate <cert> --private-key <key>`
- Test EIF after build: `nitro-cli describe-eif --eif-path <output.eif>`
- Document PCR values from build output for KMS policy configuration

## Safety Ratings

```
safety_ratings:
  - "docker info: GREEN — read-only Docker status"
  - "docker images: GREEN — read-only image listing"
  - "nitro-cli build-enclave: GREEN — builds EIF file, no runtime impact"
  - "nitro-cli describe-eif: GREEN — read-only EIF inspection"
  - "docker run: YELLOW — runs container, verify image is trusted"
```

## Escalation Conditions

- Docker image contains proprietary dependencies that cannot be bundled
- EIF size exceeds available instance memory
- Build requires signing certificate that is managed by security team
- Architecture mismatch between build environment and target instance

## Data Sensitivity

- `docker inspect`: MEDIUM — may reveal image configuration and environment variables
- `nitro-cli build-enclave`: LOW — produces EIF file
- `nitro-cli describe-eif`: LOW — returns PCR values and metadata
- Docker image contents: HIGH — may contain application code and embedded credentials

## Prohibited Actions

- NEVER embed secrets or credentials in the Docker image — pass them via vsock at runtime
- NEVER build EIF from untrusted Docker images without security review
- NEVER ignore build warnings about missing entrypoint or large image size
- NEVER distribute EIF files through unsecured channels — they contain the application

## Phase 3 — Rollback

1. If build produced invalid EIF, delete it and rebuild from corrected Docker image
2. If Docker image was modified, restore from container registry
3. If signing certificate was used incorrectly, rebuild without signing or with correct certificate
4. Verify rebuilt EIF with describe-eif before deploying
5. Update KMS key policies if PCR values changed due to rebuild

## Common Issues

- symptoms: "build-enclave fails with 'Docker image not found'"
  diagnosis: "Docker image does not exist locally or Docker daemon is not running."
  resolution: "Pull the image: docker pull <image>. Ensure Docker is running: sudo systemctl start docker."

- symptoms: "EIF builds but enclave crashes immediately on start"
  diagnosis: "Docker image ENTRYPOINT/CMD is missing or the application crashes without network access."
  resolution: "Verify Dockerfile has ENTRYPOINT or CMD. Ensure application does not require network at startup."

## Output Format

```yaml
root_cause: "image_build — <specific_cause>"
evidence:
  - type: build_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Docker image and rebuild EIF"
  long_term: "Implement CI/CD pipeline for EIF builds with validation"
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
