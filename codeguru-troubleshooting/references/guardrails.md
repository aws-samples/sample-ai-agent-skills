# Amazon CodeGuru Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon CodeGuru issue.

## Guardrail 1: Repository Association Must Be ASSOCIATED
CodeGuru Reviewer requires a repository association in ASSOCIATED state. Associations can be in ASSOCIATING, ASSOCIATED, DISASSOCIATING, or FAILED states. Reviews only work when ASSOCIATED.

## Guardrail 2: Pull Request Reviews Are Automatic After Association
Once associated, CodeGuru Reviewer automatically analyzes pull requests. However, full repository scans must be explicitly triggered via CreateCodeReview API.

## Guardrail 3: Profiler Agent Must Be Integrated Into Application
CodeGuru Profiler requires the profiler agent library integrated into the application code. The agent runs within the application process and sends profiling data to AWS. Without agent integration, no data is collected.

## Guardrail 4: Profiler Data Has 5-15 Minute Delay
After the profiler agent starts, data takes 5-15 minutes to appear in the console. The agent collects samples and batches them before sending. Do not expect real-time data.

## Guardrail 5: Security Detector Is Separate From Code Quality
Security findings (vulnerabilities, secrets) are separate from code quality recommendations. They use different detection engines. Both can be enabled independently.

## Guardrail 6: Reviewer Supports Specific Languages
CodeGuru Reviewer supports Java and Python for code quality recommendations. Security detector supports additional languages. Unsupported languages produce no recommendations.

## Guardrail 7: Profiler Supports Java and Python
CodeGuru Profiler agent is available for Java (JVM) and Python applications. Other runtimes are not supported. The agent adds minimal overhead (<1% CPU).

## Guardrail 8: Suppressed Recommendations Are Hidden Not Deleted
Suppressing a recommendation hides it from the default view but does not delete it. Suppressions can be reversed. Suppression rules can be broad or specific.

## Guardrail 9: Full Repository Scans Have Costs
Full repository scans (RepositoryAnalysis) are charged per scan. Pull request reviews are charged per line of code. Monitor costs for large repositories.

## Guardrail 10: Repository Provider Configuration Varies
GitHub, GitHub Enterprise, Bitbucket, and CodeCommit have different association requirements. GitHub requires a connection (CodeStar Connections). CodeCommit uses IAM.

## Guardrail 11: Profiler Anomaly Detection Requires Baseline
Anomaly detection compares current behavior against a baseline. New profiling groups need time to establish a baseline. Anomalies are not detected immediately.

## Guardrail 12: Recommendations Have Confidence Levels
Reviewer recommendations include confidence levels. Low-confidence recommendations may be false positives. Review recommendations in context before acting.
