# Amazon GameLift Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any GameLift issue.

## Guardrail 1: Two Fleet Types with Different Capabilities
Managed EC2 fleets have GameLift managing instances, scaling, and health checks. Anywhere fleets use your own compute (on-premises, other clouds). Troubleshooting paths differ significantly. Always identify fleet type first.

## Guardrail 2: Builds and Scripts Are Not Interchangeable
Builds contain compiled game server executables for custom game servers. Scripts contain JavaScript for Realtime Servers. A build cannot be used with Realtime Servers and a script cannot be used with custom game server fleets.

## Guardrail 3: Server SDK Integration Is Required
Custom game servers MUST integrate the GameLift Server SDK. The server must call ProcessReady(), report health via onHealthCheck(), and accept game sessions via onStartGameSession(). Missing SDK calls cause health check failures and fleet activation issues.

## Guardrail 4: Fleet Activation Requires Healthy Server Processes
A fleet transitions from ACTIVATING to ACTIVE only when at least one server process reports healthy. If no process reports healthy within the timeout, the fleet enters ERROR state. Check server process logs, not just fleet events.

## Guardrail 5: Game Session Placement Uses Queues Not Direct Fleet Access
Game session placement should use queues for resilience. Queues can span multiple fleets and regions. Direct fleet placement (CreateGameSession) bypasses queue benefits like failover and latency-based routing.

## Guardrail 6: FlexMatch Has Two Modes
FlexMatch standalone mode works without GameLift hosting (you handle placement). FlexMatch with GameLift integration automatically places matched players into game sessions. Configuration requirements differ between modes.

## Guardrail 7: Spot Fleets Can Be Interrupted
Spot instances can be reclaimed by AWS with a 2-minute warning. Game servers must handle SIGTERM gracefully. Always pair Spot fleets with On-Demand fleets in queues for fallback capacity.

## Guardrail 8: Scaling Policies React to Metrics
Auto-scaling policies (target-based or rule-based) react to CloudWatch metrics. They do not predict future demand. There is a delay between metric change and scaling action. Manual scaling is immediate but does not adapt.

## Guardrail 9: Instance Types Affect Game Performance
Not all EC2 instance types are available for GameLift. Instance type determines CPU, memory, and network performance. Changing instance type requires creating a new fleet — it cannot be changed on an existing fleet.

## Guardrail 10: Runtime Configuration Defines Server Processes
The runtime configuration specifies how many server processes run per instance, their launch paths, and parameters. Incorrect runtime configuration is a common cause of fleet activation failures.

## Guardrail 11: VPC Peering Is Optional but Requires Setup
GameLift fleets can be VPC-peered with your VPC for private connectivity. VPC peering requires a VPC peering connection request and acceptance. It is not automatic and must be configured per fleet.

## Guardrail 12: Game Session Logs Require Server SDK Upload
Game session logs are only available if the server process uses the GameLift Server SDK to specify log paths. Logs are uploaded to GameLift after the game session ends. They are not available in real-time during the session.
