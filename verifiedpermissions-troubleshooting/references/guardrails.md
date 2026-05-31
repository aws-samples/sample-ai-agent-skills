# Amazon Verified Permissions Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Verified Permissions issue.

## Guardrail 1: Cedar Policy Language Is NOT IAM JSON
Verified Permissions uses Cedar policy language exclusively. Cedar uses `permit` and `forbid` keywords with `when`/`unless` clauses. Do NOT suggest IAM JSON policy syntax. Cedar entities, actions, and resources are defined by your schema, not by AWS service namespaces.

## Guardrail 2: Schema Defines Your Application's Authorization Model
The schema defines entity types, actions, and their relationships for YOUR application. It is NOT an AWS resource policy or IAM schema. Entity types represent your app's principals and resources (e.g., User, Document). Actions represent your app's operations (e.g., viewDocument, editDocument).

## Guardrail 3: Authorization Evaluates Cedar Policies, NOT IAM
`is-authorized` and `is-authorized-with-token` evaluate Cedar policies in the policy store. They do NOT evaluate IAM policies. A DENY result means no Cedar `permit` matched or a `forbid` matched. IAM permissions control who can CALL the API, not what the API decides.

## Guardrail 4: Explicit Deny Always Wins Over Permit
If any `forbid` policy matches the request, the result is DENY regardless of `permit` policies. If no `permit` matches, the result is also DENY (implicit deny). This is similar to IAM but operates on Cedar policies, not IAM policies.

## Guardrail 5: Identity Sources Map Token Claims to Cedar Entities
Identity sources (Cognito, OIDC) map JWT token claims to Cedar principal entities and attributes. The mapping is configured when creating the identity source. Token claims become entity attributes accessible in Cedar policies. Incorrect mapping causes authorization failures.

## Guardrail 6: Policy Templates Use Placeholders, NOT Variables
Policy templates use `?principal` and `?resource` as placeholders. Template-linked policies provide concrete values for these placeholders. Templates are NOT parameterized policies with runtime variables. Placeholder values are set when creating the template-linked policy, not at authorization time.

## Guardrail 7: Batch Authorization Has a 30-Request Limit
`batch-is-authorized` accepts a maximum of 30 authorization requests per call. Each request is evaluated independently. Exceeding the limit returns a validation error. For higher throughput, make multiple batch calls.

## Guardrail 8: Policy Store Is Regional, NOT Global
Each policy store exists in a single AWS region. Policies, schemas, and identity sources are regional. For multi-region applications, you must create and sync policy stores in each region. There is no built-in cross-region replication.

## Guardrail 9: Schema Validation Is Optional but Recommended
A policy store can operate without a schema. Without a schema, Cedar policies are not validated against entity types. This means typos in entity types or action names won't be caught. Always define a schema for production policy stores.

## Guardrail 10: Cedar Entities Use Hierarchical Namespaces
Cedar entity types use `Namespace::Type` format (e.g., `MyApp::User`, `MyApp::Document`). The namespace groups related entity types. Actions also use namespaces: `MyApp::Action::"viewDocument"`. Incorrect namespaces cause entity type mismatch errors.

## Guardrail 11: is-authorized-with-token Requires Identity Source
`is-authorized-with-token` only works if an identity source is configured in the policy store. The token must come from the configured identity source (Cognito user pool or OIDC provider). Using a token from an unconfigured provider returns an error.

## Guardrail 12: Policy Store Deletion Is Permanent
Deleting a policy store removes all policies, schema, identity sources, and policy templates. There is no soft-delete or recovery. Always back up policies before deletion. Use `list-policies` and `get-policy` to export policies before deleting a store.
