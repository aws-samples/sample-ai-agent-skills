---
name: textract-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Textract problems
  by analyzing document analysis, form extraction, table extraction,
  async jobs, custom queries, expense analysis, identity document
  analysis, lending document analysis, and following structured runbooks.
  Activate when: document analysis errors, form extraction failures,
  table extraction issues, async job failures, query-based extraction
  problems, expense analysis errors, or the user says something is
  wrong with Textract without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Textract, S3, IAM, CloudWatch,
  CloudTrail, and optionally SNS permissions for async job notifications.
---

# Amazon Textract Diagnostics

## When to use

Any Amazon Textract investigation where the console alone is insufficient — document analysis, form/table extraction, async processing, custom queries, expense analysis, or identity document analysis.

## Investigation workflow

### Step 1 — Collect and triage

```
aws textract detect-document-text --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}'
aws textract get-document-analysis --job-id <job-id>
aws textract get-expense-analysis --job-id <job-id>
```

### Step 2 — Domain deep dive

```
aws textract analyze-document --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["FORMS","TABLES"]'
aws textract analyze-document --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["QUERIES"]' --queries-config '{"Queries":[{"Text":"What is the total?"}]}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=textract.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Textract --metric-name SuccessfulRequestCount --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

Read `references/guardrails.md` before concluding on any Textract issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `textract detect-document-text` | Extract raw text from document |
| `textract analyze-document` | Extract forms, tables, queries |
| `textract start-document-analysis` | Async multi-page analysis |
| `textract get-document-analysis` | Get async job results |
| `textract analyze-expense` | Extract expense/receipt data |
| `textract analyze-id` | Extract identity document data |
| `textract start-lending-analysis` | Analyze lending documents |
| CloudWatch metrics | Monitor API calls, errors |

## Gotchas: Amazon Textract

- Synchronous APIs (DetectDocumentText, AnalyzeDocument) support single-page documents only. Use async APIs for multi-page.
- Async APIs (Start*) support multi-page PDFs up to 3000 pages. Results retrieved with Get* APIs.
- Image size limit: 10MB for sync, stored in S3 for async. Supported formats: JPEG, PNG, PDF, TIFF.
- Queries feature requires FeatureTypes=["QUERIES"]. Queries are natural language questions about the document.
- Form extraction returns key-value pairs. Table extraction returns cells with row/column indices.
- AnalyzeExpense is specialized for receipts/invoices. AnalyzeID is for identity documents. Don't use generic APIs for these.
- Async jobs require IAM role for SNS notification. Results expire after 7 days.
- Confidence scores vary by document quality. Low-quality scans produce lower confidence.

## Anti-hallucination rules

1. Always cite specific job IDs, block types, or API responses as evidence.
2. Sync APIs are single-page only. Never assume multi-page sync processing.
3. Async results expire after 7 days. Never assume indefinite availability.
4. Queries require explicit feature type. Never assume automatic query support.
5. Expense and ID analysis use specialized APIs. Never use generic AnalyzeDocument.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Document Analysis | A1-A2 | Text detection, document analysis |
| B — Forms & Tables | B1-B2 | Form extraction, table extraction |
| C — Async Jobs | C1-C2 | Job management, multi-page processing |
| D — Queries | D1-D2 | Custom queries, query accuracy |
| E — Specialized | E1-E2 | Expense analysis, identity documents |
| F — Quality | F1-F2 | Document quality, confidence tuning |
| Z — Catch-All | Z1 | General troubleshooting |
