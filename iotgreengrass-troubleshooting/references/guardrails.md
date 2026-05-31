# AWS IoT Greengrass V2 Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS IoT Greengrass V2 issue.

## Guardrail 1: Greengrass V2 Is NOT V1
Greengrass V2 uses a completely different architecture from V1. V2 uses a nucleus-based component model, not Lambda-based groups. V2 deployments target thing groups or individual devices, not Greengrass groups. V2 components use recipes and artifacts, not Lambda function definitions. Never apply V1 troubleshooting steps to V2.

## Guardrail 2: Nucleus Is the Core Runtime
The Greengrass nucleus manages all component lifecycle, IPC, and cloud connectivity. If the nucleus is unhealthy, no components work. Check nucleus logs first at /greengrass/v2/logs/greengrass.log. Nucleus version must be compatible with deployed component requirements.

## Guardrail 3: Component Recipes Must Be Valid
Component recipes (YAML or JSON) define lifecycle scripts, dependencies, configuration, and artifacts. Invalid recipes cause deployment failures. The recipe must specify the correct component version, publisher, and platform. Lifecycle sections (install, run, startup, shutdown) must use valid script syntax.

## Guardrail 4: Token Exchange Role Provides AWS Credentials
The token exchange service (TES) uses the device certificate to obtain temporary AWS credentials via an IAM role. The TES role must have permissions for every AWS service that components access (S3, Kinesis, IoT, etc.). Missing permissions cause silent failures in components. Check the role alias and IAM role policies.

## Guardrail 5: MQTT Bridge Requires Explicit Topic Mapping
The MQTT bridge component does NOT automatically forward all messages between local MQTT and cloud IoT Core. You must configure explicit topic mappings specifying source, target, and direction (LocalMqtt to IotCore or vice versa). Missing mappings mean messages are silently dropped.

## Guardrail 6: Stream Manager Has Local Storage Limits
Stream manager stores data locally before exporting to cloud services. Default storage is 256 MB. When storage is full, the oldest data is overwritten (or new data is rejected based on policy). Export failures accumulate local data. Monitor disk usage on the core device.

## Guardrail 7: Local Shadow Service Uses IPC Not MQTT
Components interact with local shadows via IPC (inter-process communication), not MQTT. The local shadow service is a separate component (aws.greengrass.ShadowManager). It must be deployed explicitly. Cloud shadow sync is optional and must be configured per shadow.

## Guardrail 8: OTA Deployments Use Thing Group Targeting
Deployments target either a specific core device or a thing group. Thing group deployments apply to all core devices in the group. Multiple deployments can target the same device — they are merged. Deployment conflicts occur when multiple deployments specify different versions of the same component.

## Guardrail 9: Component Dependencies Must Be Explicit
Components must declare dependencies in their recipe. The nucleus resolves dependencies and deploys them in order. Circular dependencies cause deployment failures. Hard dependencies prevent the component from starting if the dependency fails. Soft dependencies allow the component to start regardless.

## Guardrail 10: Artifacts Must Be Accessible
Component artifacts (binaries, scripts, Docker images) must be stored in S3 and accessible via the TES role. The artifact URI in the recipe must be correct. Large artifacts may cause deployment timeouts. Docker artifacts require the Docker application manager component.

## Guardrail 11: Device Certificate Must Be Valid for IoT Core
The core device uses an X.509 certificate for authentication with IoT Core. The certificate must be ACTIVE, attached to the thing, and have an IoT policy allowing greengrass operations. Certificate expiry or revocation breaks all cloud connectivity including deployments.

## Guardrail 12: Rollback Configuration Determines Failure Behavior
Deployment rollback configuration specifies what happens when a deployment fails. Options include rolling back to the previous configuration or leaving the failed state. Without rollback configuration, failed deployments leave components in a broken state. Always configure rollback for production deployments.
