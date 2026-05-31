---
name: iotgreengrass-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT Greengrass V2
  problems by analyzing core device setup, component deployments, Lambda
  components, MQTT bridge, stream manager, local shadow service, OTA
  updates, IPC communication, and edge runtime issues using structured
  runbooks. Activate when: core device provisioning failures, component
  deployment errors, Lambda component crashes, MQTT bridge connectivity,
  stream manager export failures, local shadow sync issues, OTA update
  failures, nucleus errors, or the user says something is wrong with
  Greengrass without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Greengrass, IoT, IAM, CloudWatch,
  S3, and Lambda permissions as applicable.
---

# AWS IoT Greengrass V2 Diagnostics

## When to use

Any AWS IoT Greengrass V2 investigation — core device provisioning, component deployment, Lambda components, MQTT bridge, stream manager, local shadow, OTA updates, nucleus configuration, or edge runtime troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws greengrassv2 list-core-devices --status HEALTHY
aws greengrassv2 get-core-device --core-device-thing-name <thing-name>
aws greengrassv2 list-deployments --target-arn <thing-or-group-arn>
aws greengrassv2 get-deployment --deployment-id <id>
```

### Step 2 — Domain deep dive

```
aws greengrassv2 list-installed-components --core-device-thing-name <thing-name>
aws greengrassv2 list-effective-deployments --core-device-thing-name <thing-name>
aws greengrassv2 describe-component --arn <component-arn>
aws logs filter-log-events --log-group-name /aws/greengrass/GreengrassSystemComponent/<region>/<account> --start-time <epoch>
```

### Step 3 — Detailed investigation

```
aws greengrassv2 list-component-versions --arn <component-arn>
aws iot describe-thing --thing-name <core-device>
aws iot list-thing-principals --thing-name <core-device>
aws cloudwatch get-metric-statistics --namespace AWS/Greengrass --metric-name ConnectedDevices --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any Greengrass issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-core-device` | Check core device status and connectivity |
| `get-deployment` | Check deployment status and errors |
| `list-installed-components` | Verify components on a core device |
| `list-effective-deployments` | Check which deployments apply |
| `describe-component` | Get component recipe details |
| Greengrass logs | Check nucleus and component logs on device |
| CloudWatch Metrics | Check connectivity and component health |

## Gotchas: AWS IoT Greengrass V2

- Greengrass V2 nucleus is the core runtime. It manages component lifecycle, IPC, and cloud connectivity. Nucleus version must be compatible with component requirements.
- Components have recipes (YAML/JSON) and artifacts. Recipes define lifecycle, dependencies, and configuration. Artifacts are the actual binaries/scripts. Both must be correct for deployment.
- Token exchange service provides AWS credentials on the core device. It uses the device certificate to get temporary credentials via an IAM role. The role must have permissions for all AWS services components need.
- MQTT bridge connects local MQTT to cloud IoT Core. It requires explicit topic mapping configuration. Messages don't automatically flow between local and cloud.
- Stream manager exports data to S3, Kinesis, or IoT Analytics. It has local storage limits and export retry policies. Stream manager is an optional component.
- Local shadow service syncs device shadows locally. It's separate from cloud shadow sync. Components use IPC to interact with local shadows.
- OTA updates use deployments targeting thing groups or individual devices. Rollback configuration determines behavior on failure.

## Anti-hallucination rules

1. Always cite specific deployment IDs, component versions, or device logs as evidence.
2. Greengrass V2 is fundamentally different from V1. Never mix V1 and V2 concepts.
3. Component dependencies must be explicitly declared. Never assume implicit dependencies.
4. Token exchange role permissions must match component needs. Never assume broad permissions.
5. MQTT bridge requires explicit topic mapping. Never assume automatic message routing.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Core Device | A1-A3 | Provisioning failures, nucleus errors, connectivity |
| B — Deployment | B1-B3 | Deployment failures, component conflicts, rollback |
| C — Lambda | C1-C2 | Lambda component errors, runtime issues |
| D — MQTT Bridge | D1-D2 | Bridge configuration, topic mapping |
| E — Stream Manager | E1-E2 | Export failures, storage issues |
| F — Shadow | F1-F2 | Local shadow sync, cloud shadow sync |
| G — OTA Updates | G1-G2 | Update failures, rollback issues |
| Z — Catch-All | Z1 | General troubleshooting |
