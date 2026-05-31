# Amazon Textract Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Textract issue.

## Guardrail 1: Synchronous APIs Are Single-Page Only
DetectDocumentText and AnalyzeDocument (sync) process only the first page of multi-page documents. For multi-page PDFs/TIFFs, use async APIs (StartDocumentTextDetection, StartDocumentAnalysis). Sync APIs accept S3 objects or base64 bytes.

## Guardrail 2: Async Jobs Require S3 Input
Async APIs (Start*) only accept documents from S3. Documents cannot be passed as bytes. S3 bucket must be in the same region as the API call. Results are retrieved with Get* APIs using the JobId.

## Guardrail 3: Async Results Expire After 7 Days
Results from async jobs are available for 7 days after completion. After expiration, Get* APIs return InvalidJobIdException. Store results immediately after job completion if long-term access is needed.

## Guardrail 4: Queries Require Explicit Feature Type
The Queries feature must be explicitly enabled by including "QUERIES" in FeatureTypes. Queries are natural language questions. Maximum 15 queries per AnalyzeDocument call. Queries work best with structured documents.

## Guardrail 5: Form Extraction Returns Key-Value Pairs
AnalyzeDocument with FORMS feature returns KEY_VALUE_SET blocks. Keys and values are linked by relationship IDs. Not all documents have clear key-value structure. Handwritten forms may have lower accuracy.

## Guardrail 6: Table Extraction Returns Cell-Based Structure
AnalyzeDocument with TABLES feature returns TABLE, CELL blocks with row/column indices. Merged cells are represented with spanning indices. Complex table layouts may not be perfectly extracted.

## Guardrail 7: Expense Analysis Uses Specialized API
AnalyzeExpense is optimized for receipts and invoices. It extracts standardized fields (vendor, total, line items). Don't use generic AnalyzeDocument for expense documents — AnalyzeExpense provides better structured output.

## Guardrail 8: Identity Document Analysis Uses Specialized API
AnalyzeID extracts fields from identity documents (driver's licenses, passports). It returns standardized fields. Don't use generic AnalyzeDocument for ID documents.

## Guardrail 9: Document Quality Directly Affects Accuracy
Low-resolution scans, skewed documents, poor lighting, and handwriting all reduce accuracy. Confidence scores reflect extraction quality. Preprocess documents (deskew, enhance) for better results.

## Guardrail 10: Supported Formats Are Limited
Textract supports JPEG, PNG, PDF, and TIFF. PDF and TIFF support multi-page (async only). Other formats must be converted first. Password-protected PDFs are not supported.

## Guardrail 11: Multi-Page PDFs Have Page Limits
Async APIs support up to 3000 pages per document. Very large documents may take significant processing time. Consider splitting extremely large documents.

## Guardrail 12: Lending Analysis Is Purpose-Built
StartLendingAnalysis is specifically designed for mortgage lending documents. It classifies and extracts from lending-specific document types. Don't use for general document analysis.
