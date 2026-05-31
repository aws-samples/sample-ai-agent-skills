# Amazon Monitron Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Monitron issue.

## Guardrail 1: Hardware and Software Are Interdependent
Monitron uses physical gateways and sensors. Many issues require physical intervention — repositioning sensors, power-cycling gateways, or checking WiFi signal strength. API-only troubleshooting cannot resolve hardware problems.

## Guardrail 2: 14-Day Learning Period Is Mandatory
New sensors require at least 14 days of continuous data collection before the ML model can reliably detect anomalies. Alerts during this period are unreliable. Do not treat early alerts as confirmed anomalies or dismiss the system as broken.

## Guardrail 3: Mobile App Is Required for Device Setup
Initial gateway commissioning and sensor pairing MUST be done through the Monitron mobile app. There is no CLI or API alternative for physical device setup. The app uses Bluetooth to communicate with devices.

## Guardrail 4: WiFi Requirements Are Specific
Gateways require 2.4 GHz WiFi (not 5 GHz), WPA2 security, and outbound internet access to AWS endpoints. Enterprise WiFi with captive portals or certificate-based authentication may not work without additional configuration.

## Guardrail 5: BLE Range Limits Sensor-Gateway Communication
Sensors communicate with gateways via Bluetooth Low Energy (BLE) with approximately 20-30 meter range with line of sight. Walls, metal structures, and industrial equipment significantly reduce effective range.

## Guardrail 6: Sensor Data Gaps May Be Normal
Brief data gaps (minutes) can occur during gateway reconnection or sensor battery replacement. Gaps longer than 1 hour indicate connectivity issues. Gaps longer than 24 hours indicate sensor or gateway failure.

## Guardrail 7: Inference Results Depend on Mounting Position
Sensor mounting position and orientation affect vibration and temperature readings. Incorrect mounting produces unreliable data and false anomalies. Sensors must be mounted per Monitron guidelines on the bearing housing.

## Guardrail 8: Projects Are Regional
Monitron projects are bound to a specific AWS region. Gateways and sensors registered to a project operate in that region. Cross-region project management is not supported.

## Guardrail 9: User Management Uses AWS SSO / IAM Identity Center
Monitron user access is managed through AWS IAM Identity Center (formerly AWS SSO). Users must be provisioned in IAM Identity Center before they can be added to Monitron projects. Direct IAM user access is not supported.

## Guardrail 10: Data Export Has Specific Format
Monitron data export produces files in a specific format (not standard CSV or JSON). Exported data includes vibration, temperature, and inference results. Custom parsing is required to use exported data in external systems.

## Guardrail 11: Gateway Firmware Updates Are Automatic
Gateway firmware updates are managed by AWS and applied automatically. There is no manual firmware update mechanism. If a gateway is offline during an update window, it updates when reconnected.

## Guardrail 12: Sensor Battery Life Affects Data Collection
Monitron sensors are battery-powered with approximately 3-year battery life under normal conditions. Increased measurement frequency or poor BLE connectivity reduces battery life. Dead batteries cause permanent data gaps until replaced.
