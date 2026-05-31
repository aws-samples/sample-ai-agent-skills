# Amazon QLDB Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon QLDB issue.

## Guardrail 1: QLDB Journal Is Immutable
The journal is an append-only, immutable log. You cannot delete, modify, or overwrite journal entries. DELETE operations mark documents as deleted but the history remains. This is by design for audit and compliance.

## Guardrail 2: PartiQL Is NOT Standard SQL
QLDB uses PartiQL, which extends SQL for semi-structured data. Key differences: INSERT uses VALUE keyword, UPDATE uses SET with document syntax, no ALTER TABLE, no JOINs across tables in some contexts. Ion data types differ from SQL types.

## Guardrail 3: QLDB Uses Amazon Ion Data Format
QLDB stores data in Amazon Ion format (superset of JSON). Ion supports additional types: timestamp, blob, clob, decimal, symbol, struct, list, sexp. JSON is valid Ion but not all Ion is valid JSON. Use Ion libraries for proper data handling.

## Guardrail 4: Indexes Are Required for Efficient Queries
Queries without indexes perform full table scans. QLDB supports up to 5 indexes per table. Indexes must be created before data insertion for best performance. Index creation on existing tables requires a table scan.

## Guardrail 5: Transactions Have OCC (Optimistic Concurrency Control)
QLDB uses optimistic concurrency control. Concurrent transactions on the same document may cause OCC conflicts. Conflicting transactions are automatically retried by the driver. Implement idempotent transactions for reliability.

## Guardrail 6: Journal Export Is Asynchronous
Journal exports to S3 are asynchronous jobs. Exports include all committed transactions in the specified time range. Export format is Ion. The IAM role must have S3 write permissions. Large exports can take significant time.

## Guardrail 7: Verification Uses Cryptographic Hash Chains
QLDB provides cryptographic verification using SHA-256 hash chains. The digest represents the complete journal state. Revision proofs allow verifying individual documents. Verification requires the digest and proof — both must be obtained from QLDB.

## Guardrail 8: Streaming Delivers to Kinesis Data Streams
QLDB streams deliver journal data to Kinesis Data Streams in near real-time. Streams include BLOCK_SUMMARY and REVISION_DETAILS records. The Kinesis stream must exist before creating the QLDB stream. IAM role must have kinesis:PutRecord permission.

## Guardrail 9: Table Deletion Doesn't Delete History
Dropping a table removes it from active use but the journal history remains. All historical revisions are preserved. This is fundamental to QLDB's immutability guarantee.

## Guardrail 10: Session Limits Apply Per Ledger
Each ledger has a limit on concurrent active sessions. Exceeding the limit returns SessionPoolEmptyException. Use connection pooling in the QLDB driver. Close sessions when done.

## Guardrail 11: Document Size Has Limits
Individual documents have a maximum size (typically 4 MB). Transaction size is also limited. Large documents should be split or stored in S3 with references in QLDB.

## Guardrail 12: Ledger Deletion Has a Protection Period
Ledgers have deletion protection enabled by default. You must disable deletion protection before deleting a ledger. Deleted ledgers cannot be recovered. Export journal data before deletion.
