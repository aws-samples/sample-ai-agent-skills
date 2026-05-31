# Amazon Q Developer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Q Developer issue.

## Guardrail 1: Authentication Is Required
Q Developer requires sign-in via AWS Builder ID (free tier) or IAM Identity Center (Pro tier). Without authentication, the plugin is non-functional. Check authentication status in the IDE status bar.

## Guardrail 2: Code Suggestions Are Context-Dependent
Suggestion quality depends on the code context (open files, cursor position, comments). Insufficient context produces poor or no suggestions. Adding comments describing intent improves suggestions.

## Guardrail 3: Security Scanning Has Language Limitations
Security scanning supports a subset of languages (Python, Java, JavaScript, TypeScript, C#, Go, and others). Unsupported languages are skipped. Scan results are not exhaustive — use as one security layer.

## Guardrail 4: Code Transformation Supports Specific Migrations
Code transformation supports specific language/framework upgrades (e.g., Java 8→17). Not all migrations are supported. Results require manual review and testing. Large codebases may need multiple passes.

## Guardrail 5: Free Tier Has Monthly Limits
Builder ID (free tier) has monthly limits on code suggestions and security scans. Exceeding limits stops functionality until the next billing period. Pro tier has higher limits.

## Guardrail 6: IDE Plugin Version Matters
Plugin features and compatibility depend on the IDE and plugin version. Outdated plugins may lack features or have bugs. Keep plugins updated to the latest version.

## Guardrail 7: Customization Requires Pro Tier
Custom models (trained on your codebase) require the Pro tier with IAM Identity Center. Customization takes hours to build. Training data must meet format and quality requirements.

## Guardrail 8: Network Connectivity Required
Q Developer requires internet connectivity to AWS endpoints. Corporate firewalls or proxies may block connections. The plugin needs HTTPS access to Q Developer service endpoints.

## Guardrail 9: Suggestions May Include Open Source References
Code suggestions may reference open source code. Q Developer provides reference tracking to identify suggestions matching open source. Review reference information for licensing compliance.

## Guardrail 10: Chat and Inline Suggestions Are Different Features
Q Developer provides both inline code suggestions and chat-based assistance. They use different interaction models. Chat supports multi-turn conversations while inline suggestions are contextual completions.

## Guardrail 11: Workspace Indexing Improves Quality
Q Developer can index your workspace for better context. Indexing takes time for large projects. Without indexing, suggestions rely only on open files.

## Guardrail 12: Administrative Controls Affect User Experience
Organization administrators can configure policies that restrict features, block certain suggestion types, or require reference tracking. User-facing issues may stem from admin policies.
