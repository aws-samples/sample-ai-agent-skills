# Control Tower Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Control Tower issue.

## Guardrail 1: SCPs Do NOT Apply to the Management Account
Service Control Policies never restrict the management account, even if attached to the root or an OU containing it. The management account can always perform any action. Never claim SCPs protect the management account.

## Guardrail 2: Controls Are NOT Inherited by Child OUs
Enabling a control on a parent OU does NOT automatically apply it to child (nested) OUs. You must enable controls on each OU individually. This is a common source of compliance gaps.

## Guardrail 3: Detective Controls Detect, They Don't Prevent
Detective controls (Config rules) evaluate compliance AFTER an action occurs. They do NOT block the action. Only preventive controls (SCPs) and proactive controls (CF hooks) can prevent actions. Never claim a detective control blocks anything.

## Guardrail 4: Never Modify CT-Managed Resources Directly
Control Tower manages CloudFormation stacks, StackSets, SCPs, Config rules, and IAM roles. Modifying these directly causes DRIFT. Always use the Control Tower console or APIs to make changes. This includes stacks prefixed with AWSControlTower*.

## Guardrail 5: Resolve Drift Before Updating Landing Zone
Landing zone updates will fail if drift is present. You MUST resolve all drift (re-register OUs, move accounts back, restore SCPs) before attempting a landing zone update. Check drift status first.

## Guardrail 6: Account Factory Uses Service Catalog
Account Factory provisions accounts through AWS Service Catalog, which creates CloudFormation stacks. Provisioning failures are often CloudFormation failures. Check the Service Catalog provisioned product record AND the underlying CloudFormation stack events.

## Guardrail 7: Drift Resolution Order Matters
Resolve drift in this order: (1) SCP drift, (2) OU drift, (3) account drift, (4) landing zone drift. Resolving in the wrong order can cause cascading failures or incomplete resolution.

## Guardrail 8: OU Registration is Required for Governance
Simply creating an OU in Organizations does NOT bring it under Control Tower governance. You must REGISTER the OU with Control Tower. Unregistered OUs have no controls, no Config rules, and no StackSet deployments.

## Guardrail 9: CT IAM Roles Are Critical Infrastructure
Control Tower creates roles like AWSControlTowerExecution, AWSControlTowerAdmin, AWSControlTowerCloudTrailRole, and AWSControlTowerConfigAggregatorRoleForOrganizations. Deleting or modifying these roles breaks CT operations and causes drift.

## Guardrail 10: Nested OUs Have Limited Support
Nested OUs (up to 5 levels) are supported, but each level must be individually registered and have controls enabled separately. StackSet deployments target registered OUs only. Deeply nested structures increase operational complexity.

## Guardrail 11: Region Deny Guardrail Has Exceptions
The region deny SCP (CT.MULTISERVICE.PV.1) blocks API calls in non-governed regions but has built-in exceptions for global services (IAM, STS, Organizations, etc.). Adding custom exceptions requires modifying the SCP, which causes drift if done directly.

## Guardrail 12: Organization Trail is Shared
Control Tower creates an organization-level CloudTrail trail that logs events from ALL accounts. Creating additional organization trails causes duplicate log delivery and increased S3/CloudWatch costs. Member account trails for specific use cases are fine.
