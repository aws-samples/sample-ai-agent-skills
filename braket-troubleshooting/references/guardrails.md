# Amazon Braket Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Braket issue.

## Guardrail 1: QPU Devices Have Availability Windows
QPU (quantum processing unit) devices are not always available. Each device has specific availability windows that change regularly. Tasks submitted outside windows are queued until the next window. Check device status with `aws braket get-device` before diagnosing timeout issues.

## Guardrail 2: Simulators and QPUs Are Fundamentally Different
Local simulators, on-demand simulators (SV1, DM1, TN1), and QPU devices have different capabilities, pricing, qubit limits, and gate support. A circuit that works on a simulator may fail on a QPU due to connectivity constraints, gate set differences, or qubit count limits.

## Guardrail 3: Braket Does Not Provide Quantum Error Correction
Amazon Braket provides access to noisy intermediate-scale quantum (NISQ) devices. There is no built-in quantum error correction or error mitigation. Results from QPU devices include noise and errors inherent to the hardware. Never claim Braket corrects quantum errors.

## Guardrail 4: Results Are Stored in S3
Quantum task results are written to an S3 bucket specified at task creation. If the bucket does not exist, permissions are wrong, or the path is misconfigured, the task may complete but results will be inaccessible. Always verify S3 output configuration.

## Guardrail 5: Pricing Varies Dramatically Between Devices
Simulator pricing is per-minute of simulation time. QPU pricing is per-task (fixed) plus per-shot. Different QPU providers (IonQ, Rigetti, IQM, QuEra) have different per-task and per-shot rates. Always check current pricing before recommending device choices.

## Guardrail 6: Circuit Constraints Are Device-Specific
Each device has specific constraints: supported gate sets, qubit connectivity maps, maximum circuit depth, maximum qubit count, and maximum shots. Use `aws braket get-device` to retrieve device capabilities before validating circuits.

## Guardrail 7: Hybrid Jobs Use Classical and Quantum Resources
Hybrid jobs run classical compute (Amazon EC2 instances via SageMaker) alongside quantum device access. Both incur separate charges. Job failures can originate from either the classical or quantum component. Check both CloudWatch logs and task status.

## Guardrail 8: SDK Version Compatibility Matters
The Amazon Braket SDK (amazon-braket-sdk) must be compatible with the Braket service version. Outdated SDK versions may produce cryptic errors, fail to support new devices, or generate incompatible circuit representations. Always check SDK version first.

## Guardrail 9: Task Cancellation Has Limitations
Tasks can only be cancelled while in CREATED or QUEUED status. Once a task is RUNNING on a QPU, it cannot be cancelled. Cancelled tasks may still incur charges if they were already dispatched to the device.

## Guardrail 10: Notebook Instances Are SageMaker-Based
Braket notebook instances are Amazon SageMaker notebook instances with Braket SDK pre-installed. Issues with notebook instances may be SageMaker issues (IAM roles, VPC config, instance limits) rather than Braket-specific problems.

## Guardrail 11: Device ARN Format Is Provider-Specific
Device ARNs follow the pattern `arn:aws:braket:<region>::device/<type>/<provider>/<device-name>`. The provider segment (ionq, rigetti, iqm, quera, amazon) determines the QPU vendor. Simulator ARNs use `amazon` as the provider.

## Guardrail 12: Braket Is Regional
Amazon Braket is available in specific AWS regions. Not all devices are available in all regions. QPU device availability depends on the provider's hardware location. Verify the region supports the desired device before troubleshooting connectivity issues.
