# AWS Panorama Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Panorama issue.

## Guardrail 1: Panorama Requires Physical Appliance Hardware
AWS Panorama runs on dedicated appliance hardware (AWS Panorama Appliance). There is no software-only or virtual deployment option. The appliance contains an NVIDIA Jetson Xavier GPU for ML inference. All computer vision workloads run on-premises on the physical device.

## Guardrail 2: Device Provisioning Requires Network and Physical Access
Provisioning a Panorama appliance requires physical access to connect an Ethernet cable and USB drive with the provisioning certificate. The device must reach AWS endpoints over HTTPS (443). Provisioning creates the device identity in AWS IoT.

## Guardrail 3: Model Compilation Is Hardware-Specific
ML models must be compiled specifically for the Panorama appliance's GPU using SageMaker Neo. Models compiled for other targets (EC2, Lambda) will not work. Supported frameworks include TensorFlow, PyTorch, MXNet, and ONNX. Compilation can take 15-60 minutes.

## Guardrail 4: Camera Streams Use RTSP Protocol Only
Panorama connects to IP cameras via RTSP (Real Time Streaming Protocol). Cameras must support RTSP and be on a network reachable from the appliance. HTTP streams, ONVIF discovery, and other protocols are not directly supported. Stream URI format must be exact including credentials.

## Guardrail 5: Application Deployment Is Asynchronous and Multi-Step
Deploying an application involves downloading containers, verifying model compatibility, starting the application graph, and connecting camera streams. This process takes 10-20 minutes. Deployment status transitions through DEPLOYMENT_PENDING, DEPLOYMENT_IN_PROGRESS, and DEPLOYMENT_SUCCEEDED or DEPLOYMENT_ERROR.

## Guardrail 6: OTA Updates Can Take 30+ Minutes
Over-the-air software updates download and install new system software. Updates require stable internet connectivity throughout. Interrupting an update can leave the device in an unrecoverable state. Always verify connectivity before initiating updates.

## Guardrail 7: Device Logs Stream to CloudWatch Logs
The Panorama appliance sends application and system logs to CloudWatch Logs. Log delivery requires the device's IAM role to have logs:CreateLogGroup, logs:CreateLogStream, and logs:PutLogEvents permissions. Logs may be delayed by 1-2 minutes.

## Guardrail 8: Application Nodes Form a Directed Graph
Panorama applications are composed of nodes (camera, model, code, output) connected in a directed graph. Data flows from camera nodes through model nodes to code nodes. Incorrect graph wiring causes silent failures or no output.

## Guardrail 9: One Appliance Supports Limited Concurrent Applications
A single Panorama appliance has finite GPU memory and compute. Running too many models or applications simultaneously causes resource exhaustion. Monitor GPU utilization and memory usage through CloudWatch metrics.

## Guardrail 10: Package Import Jobs Are Required for Custom Models
Custom ML models must be packaged and imported via CreatePackageImportJob before use in applications. The import job validates the model, creates a node, and registers it. Import jobs are asynchronous and can fail due to model format issues.

## Guardrail 11: Network Configuration Requires Specific Outbound Rules
The Panorama appliance requires outbound HTTPS (443) access to multiple AWS endpoints including IoT, S3, CloudWatch, and Panorama service endpoints. Restrictive firewalls or proxies that block these endpoints cause device connectivity failures.

## Guardrail 12: Device Certificates Have Expiration
The device identity certificate provisioned during setup has an expiration date. Expired certificates cause the device to lose connectivity to AWS. Certificate rotation must be planned before expiration.
