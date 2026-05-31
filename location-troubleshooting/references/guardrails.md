# Amazon Location Service Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Location Service issue.

## Guardrail 1: Provider Capabilities Vary Significantly
Esri, HERE, GrabMaps, and OpenData providers have different capabilities, coverage, and data quality. Not all features available with all providers. Check provider-specific documentation.

## Guardrail 2: GrabMaps Covers Southeast Asia Only
GrabMaps data provider only covers Southeast Asian countries. Using GrabMaps outside this region returns no results or errors. Use Esri or HERE for global coverage.

## Guardrail 3: Place Search Results Are Provider-Dependent
Different providers have different POI (Point of Interest) databases. Search results vary by provider. The same query may return different results with different providers.

## Guardrail 4: Route Calculation Modes Vary by Provider
Car, truck, and walking travel modes are not available with all providers. Check provider support before configuring route calculators. Some providers support additional options (avoid tolls, ferries).

## Guardrail 5: Geofences Use GeoJSON Polygon Format
Geofences must be defined as GeoJSON polygons. Maximum 1000 vertices per geofence. The polygon must be closed (first and last coordinates identical). Circle geofences use center point and radius.

## Guardrail 6: Tracker-Geofence Integration Uses EventBridge
When a tracked device enters or exits a geofence, events are sent to EventBridge. The tracker must be linked to the geofence collection. EventBridge rules route events to targets (Lambda, SNS, etc.).

## Guardrail 7: API Keys Have Limited Scope
API keys provide unauthenticated access to specific resources. They are simpler than Cognito but less secure. API keys can be restricted to specific actions and resources. Use Cognito for authenticated access.

## Guardrail 8: Map Tiles Require CORS for Web Apps
Map tile requests from web browsers require proper CORS configuration. The map resource must allow the web application's origin. API keys or Cognito tokens must be included in requests.

## Guardrail 9: Position Coordinates Use [Longitude, Latitude] Order
Amazon Location Service uses [longitude, latitude] order (GeoJSON standard). This is opposite to the common [latitude, longitude] convention. Swapped coordinates cause incorrect positions.

## Guardrail 10: Tracker Position Filtering Affects Updates
Trackers can filter position updates based on accuracy, distance, or time. Filtered updates are not stored or used for geofence evaluation. Overly aggressive filtering may miss geofence events.

## Guardrail 11: Geofence Collections Have Limits
Default limit of 50,000 geofences per collection. Geofence evaluation has latency — not instant. Complex geofences (many vertices) take longer to evaluate.

## Guardrail 12: Map Styles Are Provider-Specific
Each provider offers different map styles (satellite, street, navigation). Style names are provider-specific. Not all styles available with all providers.
