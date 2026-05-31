# IAM Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any IAM issue.

## Guardrail 1: Policy Evaluation Has a Specific Order
Explicit Deny → SCP → Resource Policy → Permission Boundary → Session Policy → Identity Policy. An explicit deny at ANY level is final. Never skip layers.

## Guardrail 2: Permission Boundaries Restrict, Don't Grant
Permission boundaries set the maximum permissions an identity CAN have. They don't grant permissions. Effective permissions = intersection of boundary and identity policy. Both must allow.

## Guardrail 3: SCPs Don't Apply to Management Account
SCPs never restrict the management account or service-linked roles. Never claim SCPs protect the management account.

## Guardrail 4: Role Trust Policy is Bidirectional
Assuming a role requires BOTH: the role's trust policy allowing the principal AND the principal having sts:AssumeRole permission. Missing either side fails.

## Guardrail 5: IAM Changes Are Eventually Consistent
Policy changes may take several seconds to propagate globally. Don't assume changes are instant. Retry after a brief wait before concluding a policy change didn't work.

## Guardrail 6: Wildcard Principal Means Everyone
`"Principal": "*"` in a resource policy means ALL principals including anonymous/unauthenticated. Always use specific ARNs. Combine with Condition keys for safety.

## Guardrail 7: Condition Keys Are Case-Sensitive
IAM condition keys like aws:SourceIp, aws:PrincipalOrgID are case-sensitive. Typos in condition keys silently fail (the condition is never true), causing unexpected denials.

## Guardrail 8: MFA Conditions Don't Work with Long-Term Keys
The condition key aws:MultiFactorAuthPresent is only present in temporary credentials from STS. It's NOT present when using long-term access keys directly, even if MFA is configured on the user.

## Guardrail 9: Session Policies Only Restrict
Session policies passed during AssumeRole/GetFederationToken further restrict permissions. They cannot grant permissions beyond what the role already has.

## Guardrail 10: Resource Policies Can Grant Cross-Account
Resource-based policies (S3 bucket policy, KMS key policy, etc.) can grant cross-account access WITHOUT the identity policy allowing it. This is an exception to the normal evaluation.

## Guardrail 11: Policy Size Limits
Managed policies: 6,144 characters. Inline policies: 2,048 (user), 10,240 (role/group). Trust policies: 2,048 characters. These are hard limits after whitespace removal.

## Guardrail 12: Service-Linked Roles Are Immutable
Service-linked roles are created and managed by AWS services. You cannot modify their permissions policy. You can only delete them if the service allows it and no resources depend on them.
