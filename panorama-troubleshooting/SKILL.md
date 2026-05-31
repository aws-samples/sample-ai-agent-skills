---
name: panorama-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Panorama problems
  by analyzing device provisioning, application deployment, model
  compilation, camera stream configuration, OTA updates, networking,
  and following structured runbooks. Activate when: device provisioning
  failures, application deployment errors, model compilation issues,
  camera stream disconnections, OTA update failures, network connectivity
  problems, device offline, appliance registration errors, or the user
  says something is wrong with Panorama without naming specific symptoms.
compatibility: >
  Requires AWS CLI with Panorama, S3, IAM, CloudWatch, and IoT
  permissions. Physical Panorama appliance required for device operations.
---

# AWS Panorama Diagnostics

## When to use

Any AWS Panorama investigation — device provisioning, application deployment, model compilation, camera streams, OTA updates, or networking issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws panorama list-devices
aws panorama list-application-instances --device-id <device-id>
aws panorama describe-device --device-id <device-id>
aws panorama list-nodes
```

### Step 2 — Domain deep dive

```
aws panorama describe-application-instance --application-instance-id <id>
aws panorama list-application-instance-dependencies --application-instance-id <id>
aws panorama describe-node --node-id <node-id>
aws panorama list-package-import-jobs
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/Panorama --metric-name DeviceStatus --dimensions Name=DeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws logs filter-log-events --log-group-name /aws/panorama/devices/<device-id> --start-time <epoch-ms>
aws panorama describe-package-version --package-id <pkg-id> --package-version <ver>
```

Read `references/guardrails.md` before concluding on any Panorama issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `panorama list-devices` | List registered appliances |
| `panorama describe-device` | Get device status and details |
| `panorama list-application-instances` | List deployed apps on device |
| `panorama describe-application-instance` | Get app deployment details |
| `panorama list-nodes` | List available nodes (models, code) |
| `panorama list-package-import-jobs` | Check model import status |
| CloudWatch Logs | Device and application logs |

## Gotchas: AWS Panorama

- Panorama appliance must be on the same network subnet during provisioning. Device registration requires physical access and network connectivity to AWS.
- Model compilation targets specific hardware. Models must be compiled for the Panorama appliance's GPU (NVIDIA Jetson Xavier). SageMaker Neo compilation is required.
- Camera streams use RTSP protocol. Cameras must support RTSP and be network-accessible from the appliance. Credentials and stream URIs must be exact.
- OTA updates require stable internet. Updates can take 30+ minutes. Interrupting an update can brick the device. Ensure stable connectivity before starting.
- Application deployment is asynchronous. Deploying an app triggers model download, compilation verification, and container startup. This can take 10-20 minutes.
- Device logs are sent to CloudWatch. The appliance streams logs to CloudWatch Logs. Log delivery requires proper IAM role on the device.

## Anti-hallucination rules

1. Always cite device IDs, application instance IDs, or API responses as evidence.
2. Panorama requires physical appliance hardware. Never suggest software-only deployment.
3. Model compilation is hardware-specific. Never assume cross-platform compatibility.
4. Camera RTSP streams must be network-reachable from the appliance.
5. OTA updates are not instant. Never claim immediate update completion.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Device | A1-A2 | Device provisioning, registration errors |
| B — Applications | B1-B2 | App deployment, app lifecycle |
| C — Models | C1-C2 | Model compilation, model import |
| D — Camera | D1-D2 | Stream configuration, stream failures |
| E — Updates | E1-E2 | OTA updates, firmware issues |
| F — Network | F1-F2 | Connectivity, DNS/proxy issues |
