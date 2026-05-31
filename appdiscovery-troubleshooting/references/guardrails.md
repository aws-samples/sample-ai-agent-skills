# AWS Application Discovery Service Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Application Discovery Service issue.

## Guardrail 1: Two Discovery Methods Collect Different Data
Agent-based discovery collects detailed data including network connections, processes, and performance metrics. Agentless discovery via vCenter collects VM configuration, performance, and basic network info. Agents provide more detail for dependency mapping.

## Guardrail 2: Data Is Stored in Migration Hub Home Region
All discovery data is stored in the Migration Hub home region regardless of where agents are deployed. The home region must be set before discovery data can be viewed. Agents report to the home region endpoint.

## Guardrail 3: Agentless Collector Requires VMware vCenter
The agentless collector is a VMware OVA that connects to vCenter Server API. It only works with VMware environments. Hyper-V, KVM, and physical servers require agent-based discovery.

## Guardrail 4: Continuous Export Must Be Explicitly Enabled
Discovery data is not automatically exported to S3. Continuous export must be started via API or console. Once enabled, data flows to S3 where it can be queried with Athena.

## Guardrail 5: Agent Requires Outbound HTTPS Connectivity
Discovery agents need outbound HTTPS (port 443) to the discovery service endpoint. No inbound ports need to be opened. Proxy configuration is supported for restricted networks.

## Guardrail 6: Data Collection Must Be Started After Agent Registration
Installing and registering an agent doesn't automatically start data collection. You must explicitly start data collection via the API or console for each agent.

## Guardrail 7: Network Connection Data Requires Agent-Based Discovery
Dependency mapping relies on network connection data which is only collected by agents. Agentless discovery does not capture network connections between servers.

## Guardrail 8: Export Tasks Are One-Time Operations
Standard export tasks create a point-in-time snapshot. For ongoing data, use continuous export. Export tasks generate CSV files in S3.

## Guardrail 9: Athena Tables Are Created by Continuous Export
Athena integration requires continuous export to be active. The service creates and manages Athena tables automatically. Do not manually create or modify these tables.

## Guardrail 10: Agent Supports Specific Operating Systems
Discovery Agent supports specific OS versions (Amazon Linux, Ubuntu, RHEL, CentOS, SUSE, Windows Server). Check compatibility before deployment. Unsupported OS versions may fail silently.

## Guardrail 11: Data Retention Has Limits
Discovery data has retention limits in the service. Export data to S3 for long-term storage. Continuous export ensures ongoing data preservation.

## Guardrail 12: Tags Help Organize Discovery Data
Tags can be applied to discovered servers for organization. Tags are used in Migration Hub for application grouping. Apply tags early in the discovery process.
