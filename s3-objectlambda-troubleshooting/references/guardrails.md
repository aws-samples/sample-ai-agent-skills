# S3 Object Lambda Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any S3 Object Lambda issue.

## Guardrail 1: OLAPs Require a Supporting Access Point
Object Lambda Access Points (OLAPs) cannot be attached directly to an S3 bucket. They require a supporting standard S3 access point. The supporting access point must exist in the same region and account as the OLAP.

## Guardrail 2: WriteGetObjectResponse Is Mandatory
The Lambda function MUST call WriteGetObjectResponse (or the equivalent SDK method) to return data to the caller. If the function completes without calling this API, the S3 Object Lambda service returns a 500 Internal Server Error to the original requester. This is the most common cause of Object Lambda failures.

## Guardrail 3: 60-Second Execution Limit
S3 Object Lambda enforces a 60-second timeout for the Lambda function to call WriteGetObjectResponse. This is independent of the Lambda function's own timeout configuration. Even if the Lambda timeout is set to 15 minutes, Object Lambda will timeout at 60 seconds.

## Guardrail 4: Range and Part-Number Requests Are Forwarded
When a client sends a Range GET or part-number request through an OLAP, the request headers are forwarded to the Lambda function. The function must handle these explicitly — either by honoring the range/part-number or by returning the full object. Ignoring range headers causes incorrect behavior.

## Guardrail 5: HEAD Object Requires Separate Handling
HEAD object requests through OLAPs invoke the Lambda function with a HeadObject event type. The function must detect this event type and respond with WriteGetObjectResponse containing only headers (no body). Treating HEAD as GET wastes compute and may cause errors.

## Guardrail 6: No Anonymous Access
OLAPs do NOT support anonymous (unsigned) requests. All requests must be authenticated using SigV4. Pre-signed URLs work but must be generated for the OLAP ARN, not the underlying bucket or access point.

## Guardrail 7: Supporting Access Point Policy Matters
Both the OLAP policy AND the supporting access point policy are evaluated. A request must be allowed by both policies. A common misconfiguration is setting the OLAP policy correctly but leaving the supporting access point policy restrictive.

## Guardrail 8: Lambda Function Receives Pre-Signed URL
The Lambda function receives a pre-signed URL (inputS3Url) to fetch the original object from S3. This URL has a limited lifetime. The function must use this URL to retrieve the object — it should NOT construct its own S3 request to the bucket.

## Guardrail 9: CloudFront Requires OAC Not OAI
CloudFront integration with OLAPs requires Origin Access Control (OAC). Origin Access Identity (OAI) is NOT supported for Object Lambda origins. Using OAI results in access denied errors.

## Guardrail 10: Payload Size Limits Apply
The Lambda function response via WriteGetObjectResponse has a payload size limit. For synchronous invocation, the response must fit within Lambda's response payload limits. Very large objects may require streaming or alternative approaches.

## Guardrail 11: IAM Policies Need Both S3 and Lambda Permissions
Callers need s3-object-lambda:GetObject permission on the OLAP. The Lambda execution role needs s3-object-lambda:WriteGetObjectResponse permission AND s3:GetObject on the supporting access point. Missing any of these causes access denied.

## Guardrail 12: OLAPs Are Regional Resources
Object Lambda Access Points are regional. They can only be accessed in the region where they are created. Cross-region access is not supported. Ensure clients and CloudFront origins reference the correct regional endpoint.
