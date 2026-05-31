# AWS RoboMaker Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS RoboMaker issue.

## Guardrail 1: Simulation Runs in AWS Cloud, Not on Physical Robots
RoboMaker simulation jobs run in AWS-managed compute environments. They simulate robot behavior using Gazebo. Simulation does not deploy to or control physical robots. Physical robot deployment uses fleet management with Greengrass.

## Guardrail 2: ROS Version Compatibility Is Strict
Robot applications and simulation applications must use the same ROS version. ROS1 (Melodic) and ROS2 (Foxy) are not interchangeable. Mixing versions causes runtime failures. Check robotSoftwareSuite and simulationSoftwareSuite version fields.

## Guardrail 3: Simulation Jobs Run in Isolated VPC Environments
Simulation jobs run in AWS-managed VPCs by default. To access resources in your VPC (databases, APIs), you must configure VPC settings on the simulation job. Security groups and subnets must allow required traffic.

## Guardrail 4: Robot Applications Must Target Correct Architecture
Robot applications must be compiled for the target architecture: X86_64 for simulation, ARM64 for many physical robots. Architecture mismatch causes deployment failures. Check the application's architecture field.

## Guardrail 5: Fleet Deployment Uses AWS IoT Greengrass
Deploying robot applications to physical robots requires AWS IoT Greengrass Core installed on each robot. RoboMaker creates Greengrass deployment groups. Robots must be registered as Greengrass cores with valid certificates.

## Guardrail 6: Simulation Job Duration Has Configurable Limits
Simulation jobs have a maximum duration (default varies). Jobs are terminated when they exceed maxJobDurationInSeconds. Long-running simulations need appropriate timeout configuration. Batch simulation jobs have separate limits.

## Guardrail 7: World Generation Is Asynchronous and Resource-Intensive
WorldForge world generation jobs create 3D simulation environments from templates. Generation can take 10-60 minutes depending on complexity. Jobs consume compute resources and have concurrency limits per account.

## Guardrail 8: S3 Is Required for Application Bundles and Outputs
Robot and simulation application bundles are stored in S3. Simulation job outputs (logs, bags, videos) are written to S3. IAM roles must have appropriate S3 permissions. Incorrect S3 paths cause job failures.

## Guardrail 9: Simulation Tools Require Port Forwarding Configuration
Accessing simulation tools (Gazebo GUI, RViz, terminal) requires port forwarding configuration. These tools are not directly accessible. Use the RoboMaker console or configure port forwarding manually.

## Guardrail 10: CloudWatch Logs Are the Primary Debugging Tool
Simulation job logs are sent to CloudWatch Logs under /aws/robomaker/SimulationJobs. Robot application stdout/stderr appears in these logs. Log delivery requires IAM permissions. Logs may be delayed.

## Guardrail 11: Concurrent Simulation Job Limits Apply
Each account has limits on concurrent simulation jobs per region. Exceeding limits causes job creation failures with LimitExceededException. Check service quotas and request increases if needed.

## Guardrail 12: Robot Application Sources Must Be Valid ROS Packages
Robot applications must contain valid ROS package structure with package.xml and CMakeLists.txt (ROS1) or setup.py (ROS2). Invalid package structure causes build failures. Dependencies must be declared in the package manifest.
