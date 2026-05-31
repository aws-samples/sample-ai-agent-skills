# Amazon Managed Blockchain Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Managed Blockchain issue.

## Guardrail 1: Fabric and Ethereum Are Different Frameworks
Hyperledger Fabric uses permissioned networks with chaincode. Ethereum uses public/private networks with smart contracts. They have completely different architectures, consensus mechanisms, and programming models. Never mix concepts.

## Guardrail 2: Network Access Requires VPC Endpoints
Managed Blockchain nodes are not publicly accessible. Create a VPC endpoint for the managedblockchain service. Security groups control access. For Fabric, peer endpoints, CA endpoints, and ordering service endpoints are all VPC-only.

## Guardrail 3: Network Governance Uses Proposals
Adding members, removing members, and configuration changes require proposals. Proposals need votes from existing members. Voting policy (MAJORITY or GREATER_MAJORITY) determines approval threshold. Proposals have an expiration time.

## Guardrail 4: Chaincode Lifecycle Has Multiple Steps
Fabric chaincode: package → install on peers → approve for org → commit to channel. Each step must succeed. Chaincode must be installed on all endorsing peers. The endorsement policy determines which peers must endorse.

## Guardrail 5: Channels Provide Data Isolation
Fabric channels isolate data between subsets of members. Only channel members see channel data. Private data collections add another layer of privacy. Channel configuration changes require channel admin approval.

## Guardrail 6: Node Instance Types Determine Capacity
Node performance depends on instance type. Starter edition has limited instance types. Standard edition supports larger instances. Choose instance type based on transaction throughput and storage requirements.

## Guardrail 7: Ethereum Nodes Are Access Nodes
Managed Blockchain Ethereum provides access to mainnet/testnet via managed nodes. These are for reading data and submitting transactions. They are not mining or validator nodes. Node sync time depends on network state.

## Guardrail 8: Certificate Authority Manages Identities
Fabric networks use a Certificate Authority (CA) for identity management. Each member has a CA. User enrollment, certificate issuance, and revocation go through the CA. CA admin credentials are set during member creation.

## Guardrail 9: Ordering Service Is Managed
The Fabric ordering service is managed by AWS. You cannot configure the orderer directly. Ordering service availability affects all channels. Transaction ordering and block creation are handled automatically.

## Guardrail 10: Network Editions Have Different Limits
Starter edition is for development/testing with lower limits. Standard edition is for production with higher limits. Edition cannot be changed after network creation. Choose edition based on expected scale.

## Guardrail 11: Peer Node Logs Are in CloudWatch
Peer node logs, chaincode logs, and CA logs are available in CloudWatch Logs. Log group names follow a specific pattern. Enable logging during node creation. Logs are essential for chaincode debugging.

## Guardrail 12: Cross-Account Access Requires Invitation
Adding members from other AWS accounts requires an invitation workflow. The inviting member creates a proposal. The invited account must accept the invitation. Cross-account VPC connectivity must be configured separately.
