# Amazon VPC Lattice Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon VPC Lattice issue.

## Guardrail 1: Auth Policies Are Evaluated at Two Levels
VPC Lattice evaluates auth policies at BOTH the service network level AND the service level. Both must allow the request. A deny at either level blocks the request. When debugging auth failures, check BOTH policies. An empty auth policy (no policy set) defaults to ALLOW (auth type NONE).

## Guardrail 2: VPC Association Required for DNS Resolution
Clients can only resolve VPC Lattice service DNS names from VPCs that are associated with the service network. Without VPC association, DNS resolution fails. The association must be in ACTIVE state. Check `list-service-network-vpc-associations` to verify.

## Guardrail 3: Lambda Targets Do Not Support Health Checks
VPC Lattice does not perform health checks on Lambda targets. Lambda invocation success/failure is determined per-request. Do not configure health check settings for Lambda target groups. Health checks apply only to IP, Instance, and ALB target types.

## Guardrail 4: Access Logs Must Be Explicitly Enabled
VPC Lattice does not log access requests by default. Access log subscriptions must be created per service network. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose. Without access logs, you cannot see request details, response codes, or auth decisions.

## Guardrail 5: Service DNS Names Use Managed Prefix
VPC Lattice services get a generated DNS name (e.g., svc-xxx.vpc-lattice-svcs.us-east-1.on.aws). Traffic to this DNS resolves to link-local addresses (169.254.171.0/24 for IPv4). Custom domain names can be configured but require Route 53 alias or CNAME records pointing to the generated name.

## Guardrail 6: Target Group Type Cannot Be Changed After Creation
The target type (IP, INSTANCE, LAMBDA, ALB) is set at target group creation and cannot be modified. To change target type, create a new target group and update listener rules. Deregister targets from the old group after migration.

## Guardrail 7: Cross-Account Sharing Requires RAM Accept
Cross-account sharing uses AWS RAM. The owner shares the service network or service via RAM resource share. The consumer account must ACCEPT the RAM share (unless in the same AWS Organization with auto-accept enabled). After acceptance, the consumer associates their VPC.

## Guardrail 8: Listener Rules Evaluate in Priority Order
Rules are evaluated by priority number (lowest first). The default rule (priority 0) catches all unmatched requests. If multiple rules could match, the lowest priority number wins. Weighted target groups in a rule enable traffic splitting (blue/green, canary).

## Guardrail 9: Service Network to Service Association Required
A service must be associated with a service network to receive traffic. The association must be in ACTIVE state. A service can be associated with multiple service networks. Check `list-service-network-service-associations` to verify.

## Guardrail 10: Health Check Configuration Varies by Target Type
IP targets: HTTP/HTTPS health checks to the target directly. Instance targets: HTTP/HTTPS health checks to the instance. ALB targets: health checks go to the ALB. Each type has different default paths, ports, and protocols. Misconfigured health checks cause targets to appear unhealthy.

## Guardrail 11: VPC Lattice Uses Link-Local Addresses
Traffic to VPC Lattice services uses link-local addresses (169.254.171.0/24 for IPv4, fd00:ec2:80::/48 for IPv6). Security groups and NACLs must allow outbound traffic to these prefixes. Blocking link-local ranges breaks VPC Lattice connectivity.

## Guardrail 12: Auth Policy Syntax Is IAM-Like but Not IAM
Auth policies use IAM policy JSON syntax (Effect, Action, Resource, Condition) but are VPC Lattice resource policies, not IAM policies. They support VPC Lattice-specific condition keys like `vpc-lattice-svcs:SourceVpc` and `vpc-lattice-svcs:ServiceNetworkArn`. Do not confuse with IAM identity policies.
