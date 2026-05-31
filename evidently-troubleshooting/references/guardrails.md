# CloudWatch Evidently Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Evidently issue.

## Guardrail 1: Launches and Experiments Are Different
Launches control gradual feature rollouts with traffic percentage splits. Experiments measure the impact of variations using metrics and statistical analysis. Launches do NOT collect metrics. Experiments do NOT control rollout percentages. Do not conflate these concepts.

## Guardrail 2: Feature Evaluation Strategy Matters
Features can use client-side (local) or server-side evaluation. Client-side evaluation downloads rules to the SDK and evaluates locally — it does not call the EvaluateFeature API. Server-side evaluation calls the API for each evaluation. The strategy affects latency, cost, and offline behavior.

## Guardrail 3: Experiment Results Require Statistical Significance
Experiment results use Bayesian inference. Results are not meaningful until sufficient sample size is reached. Do not declare a winning variation based on early results with low confidence. Check the confidence interval before making conclusions.

## Guardrail 4: Variation Types Are Immutable After Creation
Feature variation types (boolean, string, long, double) cannot be changed after the feature is created. If the wrong type was chosen, the feature must be deleted and recreated. Do not suggest updating the variation type.

## Guardrail 5: Data Delivery Is Project-Level Configuration
Data delivery to S3 or CloudWatch Logs is configured at the project level, not per feature or experiment. All features and experiments in a project share the same data delivery configuration. Missing data may mean project-level delivery is not configured.

## Guardrail 6: Segments Use Pattern Matching on Entity Attributes
Segments define user subsets using pattern matching rules on entity attributes passed during evaluation. If entity attributes are not passed in the EvaluateFeature call, segment rules cannot match. Verify the application sends required attributes.

## Guardrail 7: Default Variation Serves When No Rules Match
When no launch, experiment, or override rule matches an entity, the default variation is served. Unexpected evaluation results often mean the entity does not match any active rule. Check the default variation configuration.

## Guardrail 8: Evidently Is Regional
Projects, features, launches, and experiments are regional resources. They exist in the region where they are created. Cross-region feature flags require separate projects in each region.

## Guardrail 9: Launches Have Scheduled Steps
Launches can have scheduled steps that change traffic allocation over time. If a launch appears to have wrong traffic splits, check if a scheduled step has executed or is pending. Review the launch schedule before concluding misconfiguration.

## Guardrail 10: Metric Events Are Asynchronous
Metric events sent via PutProjectEvents are processed asynchronously. There is a delay between sending events and seeing them in experiment results. Do not assume missing metrics mean the events were lost — allow processing time.

## Guardrail 11: Override Rules Take Precedence
Entity overrides (specific entity-to-variation mappings) take precedence over launch and experiment rules. If a specific entity always gets the same variation regardless of launch/experiment configuration, check for overrides.

## Guardrail 12: Stopping an Experiment Freezes Results
Stopping an experiment freezes the results at that point. No additional metric events are processed after stopping. Results cannot be recalculated. Ensure sufficient data is collected before stopping an experiment.
