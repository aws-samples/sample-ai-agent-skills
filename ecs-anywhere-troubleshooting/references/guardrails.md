# ECS Anywhere Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ECS Anywhere issue.

## Guardrail 1: EXTERNAL Launch Type Is Distinct
ECS Anywhere uses the EXTERNAL launch type, which is separate from EC2 and FARGATE. Task definitions must include EXTERNAL in requiresCompatibilities. Tasks cannot be placed on external instances using EC2 or FARGATE launch types.

## Guardrail 2: SSM Agent Is Mandatory
SSM Agent is required on all external instances for registration, credential rotation, and communication with AWS. If SSM Agent stops or loses connectivity, the ECS Agent loses its credentials and the instance appears disconnected. There is no alternative to SSM Agent for ECS Anywhere.

## Guardrail 3: Instance IDs Use mi-* Format
External instances registered through SSM use managed instance IDs (mi-*), not EC2 instance IDs (i-*). Commands and API calls must use the mi-* format. The ECS container instance ARN references the mi-* ID.

## Guardrail 4: awsvpc Network Mode Not Supported
External instances do NOT support awsvpc network mode. Only host and bridge network modes are available. Task definitions using awsvpc will fail to place on external instances. This is a fundamental limitation, not a configuration issue.

## Guardrail 5: Credentials Come Through SSM
ECS Agent on external instances receives temporary AWS credentials through SSM. These credentials are automatically rotated. If SSM Agent is not running or cannot reach AWS endpoints, credential rotation fails and tasks cannot pull images or access AWS services.

## Guardrail 6: Container Runtime Must Be Pre-Installed
Docker or containerd must be installed and running on the external instance BEFORE registering with ECS. ECS Anywhere does not install the container runtime. The ECS Agent communicates with the local container runtime to manage containers.

## Guardrail 7: Network Connectivity Requirements
External instances must have outbound HTTPS (443) connectivity to multiple AWS endpoints: ECS, SSM, ECR, CloudWatch Logs, and S3 (for ECR layer downloads). Firewall rules must allow these connections. No inbound connectivity from AWS is required.

## Guardrail 8: No GPU or Elastic Inference Support
External instances do NOT support GPU task placement constraints, Elastic Inference accelerators, or EFS volume mounts. These features are only available on EC2 launch type. Do not suggest these for ECS Anywhere workloads.

## Guardrail 9: ECS Agent Version Matters
The ECS Agent on external instances must be compatible with the ECS Anywhere features being used. Older agent versions may not support newer features. Check the agent version: the ECS Agent is installed separately from SSM Agent on external instances.

## Guardrail 10: Service Discovery Requires Network Path
Cloud Map service discovery works with ECS Anywhere, but the external instance must have network connectivity to the discovered endpoints. If external instances are on-premises and discovered services are in AWS VPCs, a VPN or Direct Connect is required.

## Guardrail 11: Task Placement Constraints Differ
External instances support attribute-based placement constraints but NOT spread or binpack placement strategies in the same way as EC2. Placement depends on available external instances with sufficient resources.

## Guardrail 12: Registration Activation Codes Expire
The SSM activation code used to register external instances has an expiration time and a registration limit. Expired or exhausted activations cannot register new instances. Create new activations as needed.
