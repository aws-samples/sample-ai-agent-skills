---
name: appstream-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon AppStream 2.0 problems
  by analyzing fleet creation, scaling issues, instance failures, image builder
  configuration, image creation, streaming session failures, latency, disconnects,
  user pool management, SAML federation, VPC configuration, internet access,
  application launch failures, app settings persistence, home folders, app
  settings storage, and following structured runbooks. Activate when: fleet
  creation failures, scaling not working, instance failures, image builder
  issues, image creation errors, streaming session failures, high latency,
  frequent disconnects, user pool problems, SAML federation errors, VPC
  configuration issues, internet access problems, application launch failures,
  app settings not persisting, home folder issues, storage problems, or the
  user says something is wrong with AppStream without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with AppStream 2.0, EC2, IAM, S3, CloudWatch,
  and CloudTrail permissions. Some operations require VPC and networking
  permissions for fleet configuration.
---

# Amazon AppStream 2.0 Diagnostics

## When to use

Any Amazon AppStream 2.0 investigation where the console alone is insufficient — fleet management, image building, streaming sessions, user authentication, VPC networking, application configuration, or storage troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws appstream describe-fleets --names <fleet-name>
aws appstream describe-stacks --names <stack-name>
aws appstream describe-image-builders --names <builder-name>
aws appstream describe-images --names <image-name>
```

### Step 2 — Domain deep dive

```
aws appstream describe-sessions --stack-name <stack> --fleet-name <fleet>
aws appstream describe-users --authentication-type USERPOOL
aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name ActualCapacity --dimensions Name=Fleet,Value=<fleet> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name InsufficientCapacityError --dimensions Name=Fleet,Value=<fleet> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appstream.amazonaws.com --max-results 20
aws s3 ls s3://<home-folder-bucket>/<user-prefix>/
aws appstream describe-fleet-metadata --fleet-name <fleet-name>
```

Read `references/appstream-guardrails.md` before concluding on any AppStream issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-fleets` | Check fleet status, capacity, configuration |
| `describe-stacks` | Check stack configuration and associations |
| `describe-image-builders` | Check image builder status |
| `describe-images` | Check image status and applications |
| `describe-sessions` | Check active streaming sessions |
| `describe-users` | Check user pool users |
| CloudWatch Metrics | Monitor capacity, sessions, errors |
| CloudTrail | Audit API calls and configuration changes |
| S3 | Check home folders and app settings |

## Gotchas: Amazon AppStream 2.0

- Fleet instances are NOT EC2 instances you manage directly. You cannot SSH into them, install software on running fleet instances, or modify them. Use image builders to create custom images with your applications.
- Image builder is for creating custom images. Connect to the image builder via the AppStream console, install applications, create the image, then use it with fleets. Image builders are temporary — delete after creating the image.
- Always-on vs on-demand fleet types: always-on instances are running and ready (faster session start, higher cost). On-demand instances start when users connect (slower start, lower cost). Choose based on user experience vs cost requirements.
- Streaming protocol is NICE DCV. It requires specific network ports (443 for HTTPS, UDP 8433 for streaming). Firewalls and proxies must allow these ports. WebSocket connections are used for the streaming session.
- User pool vs SAML: AppStream user pool is built-in user management. SAML federation integrates with external identity providers (Okta, Azure AD, etc.). They cannot be mixed on the same stack.
- Home folders use S3. Each user gets a home folder backed by an S3 bucket. The bucket is created automatically. Users can save files that persist across sessions. Home folder size affects S3 costs.
- App settings persistence uses S3. Application settings (browser bookmarks, IDE preferences) are saved to S3 and restored in new sessions. This requires enabling app settings persistence on the stack.
- Fleet auto-scaling is based on capacity utilization. Scaling policies use CloudWatch metrics. Configure minimum, maximum, and desired capacity. Scaling cooldown periods prevent rapid scaling oscillation.
- Session timeout vs disconnect timeout: idle disconnect timeout disconnects idle users. Disconnect timeout determines how long a disconnected session is preserved before termination. Max session duration limits total session length.

## Anti-hallucination rules

1. Always cite specific fleet names, image names, session IDs, or API responses as evidence.
2. Fleet instances are NOT directly manageable EC2 instances. Never suggest SSH or direct instance modification.
3. Image builders are for image creation only. Never suggest using image builders for production streaming.
4. NICE DCV is the streaming protocol. Never suggest RDP or other protocols.
5. Home folders and app settings use S3. Never suggest EBS or EFS for AppStream storage.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Fleet | A1–A3 | Fleet creation, scaling, instance failures |
| B — Image | B1–B2 | Image builder, image creation |
| C — Streaming | C1–C3 | Session failures, latency, disconnects |
| D — User | D1–D2 | User pool, SAML federation |
| E — Networking | E1–E2 | VPC config, internet access |
| F — Application | F1–F2 | App launch failures, app settings persistence |
| G — Storage | G1–G2 | Home folders, app settings |
| Z — Catch-All | Z1 | General troubleshooting |
