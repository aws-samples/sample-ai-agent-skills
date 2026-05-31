# Amazon Personalize Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Personalize issue.

## Guardrail 1: Minimum Data Requirements Are Strict
Interactions dataset requires minimum 1000 interaction records, 25 unique users, and 2 interactions per user. Below minimums, training fails or produces poor models. More data generally produces better recommendations.

## Guardrail 2: Dataset Schema Must Match CSV Exactly
Dataset import requires CSV matching the defined schema exactly. Required columns vary by dataset type: Interactions (USER_ID, ITEM_ID, TIMESTAMP), Users (USER_ID), Items (ITEM_ID). Column order must match schema.

## Guardrail 3: Solution Training Takes Hours
CreateSolutionVersion triggers model training that can take 1-6+ hours depending on data volume and recipe. Training creates an immutable solution version. Check status with DescribeSolutionVersion.

## Guardrail 4: Campaigns Are Billed While Active
Campaigns provide real-time inference endpoints. They are billed per TPS (transactions per second) provisioned, even when idle. Delete unused campaigns. MinProvisionedTPS determines minimum billing.

## Guardrail 5: Real-Time Events Take Up to 15 Minutes
Events sent via PutEvents influence recommendations within 2-15 minutes. They don't require retraining. However, new items/users may need a model retrain to be fully incorporated.

## Guardrail 6: Batch Inference Requires S3 Input/Output
Batch inference jobs read from S3 (JSON lines format) and write to S3. Input format: {"userId":"<id>"} per line for user recommendations. IAM role must have S3 access.

## Guardrail 7: Filters Have Specific Expression Syntax
Filter expressions use SQL-like syntax: INCLUDE/EXCLUDE ItemID WHERE Items.GENRE IN ("Action"). Expressions are validated at creation. Invalid syntax causes filter creation failure.

## Guardrail 8: Cold Start Requires Exploration
New users/items without interaction history need recipes with exploration (USER_PERSONALIZATION). Exploration balances known preferences with discovery. ExplorationWeight controls this balance.

## Guardrail 9: Recipe Selection Affects Recommendation Type
Different recipes serve different use cases: USER_PERSONALIZATION for user recommendations, RELATED_ITEMS for similar items, PERSONALIZED_RANKING for re-ranking. Wrong recipe produces wrong recommendation type.

## Guardrail 10: Dataset Import Is Incremental by Default
Dataset imports can be FULL (replace) or INCREMENTAL (append). Incremental adds new records without removing existing ones. Use FULL import to replace entire dataset.

## Guardrail 11: Solution Version Is Immutable
Once created, a solution version cannot be modified. To change hyperparameters or recipe, create a new solution version. Update campaign to point to new version.

## Guardrail 12: Event Tracker Required for Real-Time Events
PutEvents requires an event tracker created for the dataset group. The tracking ID from the event tracker must be included in PutEvents calls. Without it, events are rejected.
