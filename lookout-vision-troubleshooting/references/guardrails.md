# Amazon Lookout for Vision Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Lookout for Vision issue.

## Guardrail 1: Models Must Be Started Before Inference
A trained model (TRAINED status) cannot serve predictions. You must explicitly start the model (transitions to HOSTED status) before calling detect-anomalies. Starting a model takes several minutes.

## Guardrail 2: Running Models Incur Hourly Charges
Models in HOSTED status incur charges per hour. Always stop models when not actively serving predictions. Forgetting to stop a model is a common source of unexpected costs.

## Guardrail 3: Minimum Dataset Size Requirements
Training requires at least 20 normal images and 10 anomaly images. Models trained with fewer images are unreliable. More images (100+ per class) significantly improve accuracy.

## Guardrail 4: Image Quality Directly Affects Model Accuracy
Consistent lighting, camera angle, distance, and resolution are critical. Variations in image capture conditions introduce noise that degrades model performance. Standardize image capture before training.

## Guardrail 5: Labels Must Be Accurate
Mislabeled images (normal labeled as anomaly or vice versa) significantly degrade model performance. Even a small percentage of mislabeled images can cause poor accuracy. Review labels carefully before training.

## Guardrail 6: Edge Deployment Requires Greengrass V2
Deploying models to edge devices requires AWS IoT Greengrass V2. The model must be packaged as a Greengrass component using the model packaging API. Direct edge deployment without Greengrass is not supported.

## Guardrail 7: Training Is Asynchronous and Can Take Hours
Model training is an asynchronous operation that can take 30 minutes to several hours depending on dataset size. Do not assume training has failed simply because it has not completed quickly.

## Guardrail 8: Projects Are Regional
Lookout for Vision projects, datasets, and models are bound to the AWS region where they were created. Cross-region access is not supported. S3 data must be in the same region.

## Guardrail 9: Test Dataset Improves Evaluation Accuracy
Using a separate test dataset (in addition to training dataset) provides more accurate model evaluation metrics. Without a test dataset, the service uses a holdout from training data, which may overestimate performance.

## Guardrail 10: Model Versions Are Immutable
Once a model version is trained, it cannot be modified. To improve a model, you must train a new version with updated data. Previous versions remain available for comparison.

## Guardrail 11: Inference Input Must Match Training Image Characteristics
Images sent for inference should match the characteristics of training images (resolution, lighting, angle). Significantly different images produce unreliable predictions.

## Guardrail 12: S3 Bucket Must Be in the Same Region
The S3 bucket containing training images must be in the same AWS region as the Lookout for Vision project. Cross-region S3 access is not supported and causes import failures.
