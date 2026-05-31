# AWS Network Firewall Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Network Firewall issue.

## Guardrail 1: Traffic Must Be Explicitly Routed Through Firewall Endpoints
Network Firewall does NOT automatically intercept traffic. You must configure VPC route tables to send traffic to firewall endpoints (GWLBe). Without proper routing, traffic bypasses the firewall entirely. This is the #1 cause of "firewall not working." Check route tables for routes pointing to the firewall endpoint.

## Guardrail 2: Stateless Rules Evaluate BEFORE Stateful Rules
The evaluation order is: stateless rules first, then stateful rules. If a stateless rule passes or drops a packet, stateful rules never see it. Stateless rules with action "forward to stateful" send packets to the stateful engine. Use stateless for simple packet filtering; stateful for protocol-aware, connection-tracking inspection.

## Guardrail 3: Stateful Rule Order Mode Affects Evaluation
STRICT_ORDER: rules evaluated by priority number (lowest first). First matching rule wins. DEFAULT_ACTION_ORDER: pass rules evaluated first, then drop, then alert. The mode is set on the firewall policy and affects ALL stateful rule groups in that policy. Choose the mode based on your security model.

## Guardrail 4: Asymmetric Routing Breaks Stateful Inspection
Stateful rules track connections (SYN, SYN-ACK, ACK). If request traffic goes through the firewall but response traffic bypasses it (asymmetric routing), the stateful engine cannot track the connection and may drop legitimate traffic. Ensure BOTH directions of every flow traverse the SAME firewall endpoint. This is critical in multi-AZ designs.

## Guardrail 5: Firewall Subnets Must Be Dedicated
Do NOT place workloads (EC2 instances, containers, etc.) in firewall subnets. Firewall subnets should contain ONLY the firewall endpoints. Placing workloads in firewall subnets creates routing loops because the subnet's route table sends traffic to the firewall endpoint in the same subnet. Use separate subnets for workloads.

## Guardrail 6: TLS Inspection Requires CA Certificate and Client Trust
TLS inspection decrypts HTTPS traffic for inspection and re-encrypts it. This requires: (1) a CA certificate imported into ACM, (2) the CA certificate configured in the TLS inspection configuration, (3) clients must trust the CA certificate (install in trust store). Without client trust, users see certificate warnings. TLS inspection adds latency.

## Guardrail 7: Logging Must Be Explicitly Configured
Network Firewall does not log by default. Alert logs capture stateful rule matches (alert/drop actions). Flow logs capture all traffic flow records. Both must be explicitly enabled via logging configuration. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose. Without logging, you cannot see what the firewall is doing.

## Guardrail 8: Firewall Endpoints Are Per-AZ
Each firewall endpoint is in a specific AZ. Traffic should be routed to the firewall endpoint in the SAME AZ as the source. Cross-AZ routing to firewall endpoints adds latency and cost. In multi-AZ deployments, create firewall endpoints in each AZ and configure AZ-specific route tables.

## Guardrail 9: Rule Group Capacity Is Fixed at Creation
Each rule group has a capacity (number of rule units) set at creation time. Capacity CANNOT be increased after creation. If you need more rules than the capacity allows, create a new rule group with larger capacity and migrate rules. Plan capacity based on expected rule count plus growth.

## Guardrail 10: Default Actions Apply When No Rule Matches
Stateless default actions apply when no stateless rule matches a packet. Options: pass, drop, or forward to stateful. Stateful default actions (in STRICT_ORDER mode) apply when no stateful rule matches. The default action is critical — it determines what happens to unmatched traffic. A "drop all" default with no pass rules blocks everything.

## Guardrail 11: Cross-AZ Traffic Requires Careful Route Table Design
In multi-AZ VPCs, each AZ needs its own route table pointing to the local firewall endpoint. Using a single route table with one firewall endpoint causes cross-AZ traffic and potential asymmetric routing. Design route tables per-AZ with AZ-specific firewall endpoints for correct traffic flow.

## Guardrail 12: Firewall Policy References Rule Groups
A firewall policy contains references to stateless and stateful rule groups, plus default actions. Multiple firewalls can share the same policy. Rule groups can be shared across policies. Changes to a rule group affect ALL policies and firewalls referencing it. Be cautious when modifying shared rule groups.
