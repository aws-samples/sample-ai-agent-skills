---
title: "G2 — Format Conversion"
description: "Diagnose Neptune data format conversion issues"
status: active
severity: MEDIUM
triggers:
  - "format conversion"
  - "csv format"
  - "rdf format"
  - "data transform"
owner: devops-agent
objective: "Identify and resolve Neptune data format conversion issues"
context: "Neptune bulk loader requires specific formats: CSV with Neptune headers for Gremlin, and N-Triples/N-Quads/Turtle/RDF-XML for SPARQL. Converting from other formats requires transformation."
---

## Phase 1 — Triage

MUST:
- Identify source data format and target Neptune format
- Check loader error details: `curl 'https://<endpoint>:8182/loader/<load-id>?details=true&errors=true'`
- Verify CSV headers for Gremlin: ~id, ~label, ~from, ~to, property columns
- Verify RDF syntax for SPARQL data

SHOULD:
- Validate sample data against Neptune format requirements
- Check character encoding (must be UTF-8)
- Verify data types match Neptune supported types
- Check for special characters that need escaping

MAY:
- Use Neptune CSV validator tools
- Test conversion with small dataset
- Review Neptune bulk loader format documentation

## Phase 2 — Remediate

MUST:
- Transform data to correct Neptune format
- Validate transformed data before loading
- Retry loading with corrected format

SHOULD:
- Create reusable transformation scripts
- Document format requirements for data sources
- Implement format validation in data pipeline

MAY:
- Use AWS Glue for automated format transformation
- Implement continuous data integration pipeline

## Common Issues

- symptoms: "CSV missing required headers"
  diagnosis: "Gremlin CSV requires ~id and ~label columns for vertices, ~id, ~from, ~to, ~label for edges."
  resolution: "Add required Neptune CSV headers. Use ~ prefix for system columns."

- symptoms: "RDF parsing errors"
  diagnosis: "Invalid RDF syntax in N-Triples or Turtle format."
  resolution: "Validate RDF with a parser (e.g., Apache Jena riot). Fix syntax errors."

- symptoms: "Data type mismatch"
  diagnosis: "Property values do not match declared data types."
  resolution: "Use Neptune type suffixes in CSV headers (e.g., age:Int, name:String)."

## Output Format

```yaml
root_cause: "format_conversion — <specific_cause>"
evidence:
  - type: format_errors
    content: "<loader format error details>"
  - type: data_sample
    content: "<sample of problematic data>"
severity: MEDIUM
mitigation:
  immediate: "Fix data format and retry loading"
  long_term: "Implement automated format validation pipeline"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — format conversion involves handling source data
- Service Disruption Risk: LOW — conversion is a pre-processing step
- Compliance Impact: LOW — format conversion is an operational concern

## Escalation Conditions
- Format conversion produces invalid output that fails Neptune validation
- Conversion process loses data or introduces corruption
- Large-scale conversion exceeds available compute resources
- Conversion tool incompatibility with Neptune's expected format
- Character encoding issues cause data corruption during conversion

## Data Sensitivity
- Source data files — CONFIDENTIAL
- Converted output files — CONFIDENTIAL
- Conversion scripts and configurations — INTERNAL
- Data mapping specifications — INTERNAL
- Never expose source data during conversion troubleshooting

## Prohibited Actions
- Do NOT convert data without validating output against Neptune format requirements
- Do NOT discard source data after conversion until loading is verified
- Do NOT use untrusted conversion tools
- Do NOT skip character encoding verification
- Do NOT convert data without maintaining data lineage records
- Do NOT store converted files in publicly accessible locations

## Phase 3 — Rollback
- Restore original source data files if conversion corrupted them
- Revert to previous conversion scripts if new ones produce errors
- Re-run conversion with corrected parameters
- Verify converted data matches source data integrity
- Document conversion issues and format requirements
- Update conversion procedures based on findings

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
