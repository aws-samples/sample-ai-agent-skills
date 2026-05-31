# AWS IoT Events Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS IoT Events issue.

## Guardrail 1: Detector Models Are State Machines
Each detector model defines states, transitions, and actions. States have onEnter, onInput, and onExit events. Transitions are evaluated on each input message. The initial state is defined in the model. Understanding the state machine is essential for troubleshooting.

## Guardrail 2: Input Schema Must Match Payload
Input definitions specify the expected JSON structure. Attribute paths must match the payload exactly (case-sensitive). Missing required attributes cause the input to be silently dropped — no error is returned. Always validate input payloads against the schema.

## Guardrail 3: Each Key Value Creates an Isolated Instance
Detector instances are created per unique key value. Each instance has its own state, variables, and timers. Instances are completely isolated — no shared state. The key is extracted from the input payload using the path defined in the model.

## Guardrail 4: Actions Require IAM Role Permissions
All actions (SNS, SQS, Lambda, IoT publish, etc.) execute using an IAM role. The role must have permissions for each action type. Missing permissions cause silent action failures. The role trust policy must allow iotevents.amazonaws.com to assume it.

## Guardrail 5: Alarm Models Have Predefined States
Alarm models use a fixed state machine: DISABLED, NORMAL, ACTIVE, ACKNOWLEDGED, LATCHED, SNOOZE_DISABLED. You cannot add custom states. For custom state machines, use detector models. Alarm state transitions follow predefined rules.

## Guardrail 6: Condition Expressions Use IoT Events Syntax
Conditions use $input.inputName.attributePath for input values, $variable.variableName for variables, and standard operators. This is NOT SQL. String comparisons use ==. Logical operators are && and ||. Type mismatches in comparisons cause evaluation failures.

## Guardrail 7: Timers Have Minimum 60-Second Duration
Timer minimum is 60 seconds. Timer precision is approximately ±5 seconds. Timers are evaluated when input arrives, not continuously. If no input arrives, timer expiration detection is delayed. Each detector instance has its own timers.

## Guardrail 8: Variables Are Scoped to Detector Instance
Variables set in one detector instance are not visible to other instances. Variables persist across state transitions within the same instance. Variable types are inferred from the first assignment. Type changes can cause evaluation errors.

## Guardrail 9: Input Payload Size Has Limits
Input payloads have a maximum size (typically 128 KB). Oversized payloads are rejected. The payload must be valid JSON. Binary data must be base64-encoded as a string attribute. Batch input operations have separate limits.

## Guardrail 10: Evaluation Order Matters
Within a state, onInput events are evaluated before transitions. Multiple transitions are evaluated in order — the first matching transition is taken. Actions within an event execute asynchronously with no guaranteed order.

## Guardrail 11: Action Failures Don't Block Other Actions
If one action in an event fails, other actions still execute. Failed actions are logged in CloudWatch. There is no automatic retry for failed actions. Implement error handling in target services (Lambda DLQ, SQS DLQ, etc.).

## Guardrail 12: Deleting a Model Deletes All Instances
Deleting a detector model deletes all running detector instances. This is irreversible. To preserve state, export detector data before deletion. Updating a model version does NOT delete existing instances — they continue on the old version.
