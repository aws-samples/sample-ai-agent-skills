# Amazon Forecast Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Forecast issue.

## Guardrail 1: Dataset Import Requires Specific CSV Format
Target time series requires: item_id, timestamp, target_value columns. Related time series and item metadata have different required columns. Timestamp format must be consistent (yyyy-MM-dd HH:mm:ss recommended).

## Guardrail 2: AutoPredictor Is the Recommended Approach
AutoPredictor automatically selects the best algorithm and hyperparameters. Legacy CreatePredictor is deprecated. Use CreateAutoPredictor for new predictors. AutoPredictor supports ensembling.

## Guardrail 3: Predictor Training Takes Hours
CreateAutoPredictor triggers training that can take 1-6+ hours. Status must be ACTIVE before creating forecasts. Check with DescribeAutoPredictor.

## Guardrail 4: Forecast Generation Is Separate from Training
After predictor training, you must explicitly create a forecast (CreateForecast). Forecast generation also takes time. Forecasts are queryable after ACTIVE status.

## Guardrail 5: Forecast Horizon Must Match Predictor
The forecast horizon (how far ahead to predict) is set during predictor creation. Forecasts inherit this horizon. Cannot change horizon without creating a new predictor.

## Guardrail 6: Accuracy Metrics Indicate Model Quality
WAPE (Weighted Absolute Percentage Error), RMSE, MASE, and MAPE are available. Lower WAPE is better. Metrics are computed on backtesting windows. Poor metrics indicate data quality or model fit issues.

## Guardrail 7: What-If Analysis Requires Existing Forecast
What-if analyses model scenarios by modifying input data. They require an existing forecast as baseline. What-if forecasts show the impact of changes.

## Guardrail 8: Explainability Must Be Enabled at Predictor Creation
Explainability analysis shows feature importance. Must be enabled when creating the predictor (EnableExplainability=true). Cannot be added to existing predictors.

## Guardrail 9: Time Series Frequency Must Be Consistent
Data frequency (hourly, daily, weekly, monthly) must be consistent within a dataset. Mixed frequencies cause import or training failures. Specify DataFrequency in dataset group.

## Guardrail 10: Missing Values Affect Forecast Quality
Missing values in time series reduce forecast accuracy. Forecast can fill missing values (frontfill, backfill, middlefill) but excessive gaps degrade quality.

## Guardrail 11: Related Time Series Must Extend Into Forecast Horizon
Related time series data must include values for the forecast horizon period. Historical-only related data cannot be used for future predictions.

## Guardrail 12: Forecast Quantiles Define Prediction Intervals
Forecasts generate quantile predictions (P10, P50, P90 by default). P50 is the median forecast. P10/P90 provide prediction intervals. Custom quantiles can be specified.
