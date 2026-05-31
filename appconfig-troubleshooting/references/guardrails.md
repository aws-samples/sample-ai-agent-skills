# AppConfig Guardrails

1. Deployment strategies are immutable — cannot be modified after creation, only deleted and recreated
2. Configuration validators run before deployment — a validator failure means the config never deployed
3. AppConfig agent cache TTL is separate from deployment completion — clients may still serve old config
4. Feature flags require AWS.AppConfig.FeatureFlags type — freeform JSON profiles cannot use flag APIs
5. Hosted configuration versions are immutable — each change creates a new version number
6. Rollback triggers require CloudWatch alarms in the SAME region as the deployment
7. Extensions execute synchronously at action points — a slow extension blocks the deployment pipeline
8. AppConfig is NOT SSM Parameter Store — different APIs, different pricing, different caching behavior
9. Deployment percentage is based on the deployment strategy, not manual control during rollout
10. Configuration size limit is 1MB for hosted configurations — use S3 or SSM Document for larger configs
11. Environment-level deployment locks prevent concurrent deployments to the same environment
12. Deletion of an application cascades to all environments, profiles, and deployments within it
