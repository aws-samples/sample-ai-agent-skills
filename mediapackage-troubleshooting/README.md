# MediaPackage Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental MediaPackage problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental MediaPackage — channel configuration, origin endpoints, HLS/DASH/CMAF packaging, DRM, CDN authorization, live-to-VOD, and harvest jobs.

### Activate When

- Channel creation or ingest failures
- Origin endpoint configuration errors
- HLS/DASH/CMAF packaging issues
- DRM configuration or key delivery problems
- CDN authorization failures
- Live-to-VOD or harvest job errors
- MediaPackage v2 migration issues

---

## Skill Structure

```
mediapackage-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-channel-creation.md
    ├── A2-ingest-config.md
    ├── B1-origin-endpoints.md
    ├── B2-endpoint-errors.md
    ├── C1-hls-packaging.md
    ├── C2-dash-packaging.md
    ├── C3-cmaf-packaging.md
    ├── D1-drm-config.md
    ├── D2-key-rotation.md
    ├── E1-cdn-authorization.md
    ├── F1-live-to-vod.md
    ├── F2-harvest-jobs.md
    ├── G1-v2-migration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `mediapackage:*`, `mediapackagev2:*`, `cloudfront:*`, `s3:*`, `iam:GetRole`
- MediaPackage IAM role for harvest jobs

---

## License

MIT-0
