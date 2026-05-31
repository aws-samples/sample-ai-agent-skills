# AWS Ground Station Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Ground Station issue.

## Guardrail 1: Contacts Must Be Scheduled in Advance
Satellite contacts are not on-demand. You must find available pass windows using list-contacts and reserve them. Contact availability depends on satellite orbit, ground station location, and antenna availability. Reserve contacts before the pass window.

## Guardrail 2: Satellites Must Be Onboarded
Only satellites with configured satellite profiles (NORAD ID, frequency bands) and mission profiles can be contacted. Satellite onboarding requires accurate RF parameters. Incorrect parameters cause contact failures.

## Guardrail 3: Mission Profiles Define Complete Contact Configuration
Mission profiles specify tracking config, dataflow endpoints, and antenna parameters. Each parameter must match the satellite's communication specifications. Invalid parameters cause contact failures or no data delivery.

## Guardrail 4: Data Is Delivered to Your AWS Resources
Satellite data is delivered to EC2 instances (via dataflow endpoints) or S3 during the contact. No data is stored at the ground station. Dataflow endpoint EC2 instances must be running and properly configured before the contact.

## Guardrail 5: Contact Windows Are Limited by Orbital Mechanics
LEO satellites have brief pass windows (5-15 minutes) several times per day. GEO satellites have longer windows but limited coverage. Contact duration is fixed by the satellite pass. You cannot extend a contact beyond the pass window.

## Guardrail 6: Ground Stations Are at Specific Locations
AWS Ground Station antennas are at specific sites worldwide. Each site is associated with an AWS region. Not all regions have ground stations. Choose ground stations based on satellite orbit coverage needs.

## Guardrail 7: Antenna Configuration Must Match Satellite
Uplink and downlink frequencies, polarization, bandwidth, and modulation must match the satellite's communication parameters. Mismatched parameters result in no signal or corrupted data.

## Guardrail 8: Dataflow Endpoints Require EC2 Configuration
Dataflow endpoint groups define EC2 instances that receive satellite data. Instances must be in the correct VPC and subnet. Security groups must allow Ground Station traffic. Instances must be running before the contact starts.

## Guardrail 9: Contact Cancellation Has Time Limits
Contacts can be cancelled but may incur charges if cancelled too close to the start time. Check cancellation policies. Cancelled contacts release the antenna for other users.

## Guardrail 10: Tracking Configuration Affects Signal Quality
Autotrack uses the satellite signal for antenna pointing. Programmatic tracking uses predicted orbital elements. Autotrack requires sufficient signal strength. Choose tracking mode based on satellite signal characteristics.

## Guardrail 11: Wideband vs Narrowband Have Different Capabilities
Wideband antennas support higher data rates but cost more. Narrowband antennas are for lower data rate applications. Choose based on satellite data rate requirements. Not all ground stations have both types.

## Guardrail 12: Contact Status Must Be Monitored
Contact status progresses through SCHEDULING, SCHEDULED, PREPASS, PASS, POSTPASS, COMPLETED or FAILED. Monitor status via describe-contact. Failed contacts may be due to weather, equipment, or configuration issues.
