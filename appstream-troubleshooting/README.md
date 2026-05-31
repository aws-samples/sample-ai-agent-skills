# Amazon AppStream 2.0 Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon AppStream 2.0 problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon AppStream 2.0 when the console alone isn't enough — fleet management, image building, streaming sessions, user authentication, VPC networking, application configuration, and persistent storage.

### Activate When

- Fleet creation failures or configuration issues
- Fleet scaling not working as expected
- Fleet instance failures
- Image builder connection or configuration problems
- Image creation errors
- Streaming session failures or errors
- High streaming latency
- Frequent session disconnects
- User pool management issues
- SAML federation configuration errors
- VPC configuration problems for fleets
- Internet access issues from streaming sessions
- Application launch failures within sessions
- App settings not persisting across sessions
- Home folder access or storage issues
- General AppStream errors without clear symptoms

---

## Skill Structure

```
appstream-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-fleet-creation.md
    ├── A2-fleet-scaling.md
    ├── A3-instance-failures.md
    ├── B1-image-builder.md
    ├── B2-image-creation.md
    ├── C1-session-failures.md
    ├── C2-streaming-latency.md
    ├── C3-disconnects.md
    ├── D1-user-pool.md
    ├── D2-saml-federation.md
    ├── E1-vpc-config.md
    ├── E2-internet-access.md
    ├── F1-app-launch-failures.md
    ├── F2-app-settings-persistence.md
    ├── G1-home-folders.md
    ├── G2-app-settings.md
    ├── Z1-general-troubleshooting.md
    ├── appstream-guardrails.md
    └── appstream-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Fleet** | A1–A3 | Fleet creation, scaling, instance failures |
| **B — Image** | B1–B2 | Image builder, image creation |
| **C — Streaming** | C1–C3 | Session failures, latency, disconnects |
| **D — User** | D1–D2 | User pool, SAML federation |
| **E — Networking** | E1–E2 | VPC config, internet access |
| **F — Application** | F1–F2 | App launch failures, app settings persistence |
| **G — Storage** | G1–G2 | Home folders, app settings |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/appstream-guardrails.md` covering fleet instance management limitations, image builder purpose, always-on vs on-demand fleet types, NICE DCV streaming protocol, user pool vs SAML distinction, S3-backed home folders, app settings persistence, fleet auto-scaling behavior, session timeout configuration, VPC requirements, streaming endpoint requirements, and image lifecycle management.

---

## Investigation Workflow

1. **Triage** — Check fleet/stack status, image builder state, session details
2. **Deep Dive** — Examine CloudWatch metrics, session errors, user configuration
3. **Detailed** — Review CloudTrail events, S3 storage, VPC networking

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `appstream:DescribeFleets`, `appstream:DescribeStacks`, `appstream:DescribeImageBuilders`, `appstream:DescribeImages`, `appstream:DescribeSessions`, `appstream:DescribeUsers`, `cloudwatch:GetMetricStatistics`
- S3 access for home folder and app settings troubleshooting
- VPC permissions for networking issues

---

## Usage Examples

```
# Check fleet status
aws appstream describe-fleets --names my-fleet

# Check active sessions
aws appstream describe-sessions --stack-name my-stack --fleet-name my-fleet

# Check fleet capacity metrics
aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name ActualCapacity --dimensions Name=Fleet,Value=my-fleet --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 3600 --statistics Average

# Check image builder status
aws appstream describe-image-builders --names my-builder
```

---

## License

MIT-0
