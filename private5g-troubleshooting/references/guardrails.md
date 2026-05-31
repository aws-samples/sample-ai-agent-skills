# AWS Private 5G Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Private 5G issue.

## Guardrail 1: CBRS Spectrum Is US-Only
AWS Private 5G uses CBRS Band 48 (3.5 GHz) spectrum, which is only available in the United States. The service is not available in other countries. Do not suggest deploying Private 5G outside the US or using non-CBRS spectrum.

## Guardrail 2: Hardware Is Physical and Shipped by AWS
Radio units and SIM cards are physical hardware shipped by AWS. They cannot be provisioned purely through software. Radio units must be physically installed, powered on, and connected to the network before activation. Allow time for shipping and installation.

## Guardrail 3: SIM Cards Are AWS-Provided Only
AWS Private 5G uses AWS-provided SIM cards. Third-party SIM cards are not supported. Each SIM is tied to a specific network and must be activated through the Private 5G console or API. Do not suggest using SIMs from mobile carriers.

## Guardrail 4: Network Creation Requires Valid Shipping Address
Creating a network and ordering hardware requires a valid US shipping address. The address must be in a region where AWS Private 5G is available. Hardware delivery takes days to weeks — do not expect instant provisioning.

## Guardrail 5: Radio Unit Activation Is Not Instant
After physical installation, radio units must register with the Spectrum Access System (SAS) for CBRS authorization. This process can take minutes to hours. Do not assume a powered-on radio unit is immediately active.

## Guardrail 6: Device Connectivity Requires Both Radio and SIM
A device connects to the Private 5G network only when: (1) the radio unit is active and broadcasting, (2) the SIM card is provisioned and active, and (3) the device is within radio coverage range. All three conditions must be met simultaneously.

## Guardrail 7: Throughput Depends on Physical Environment
Throughput on Private 5G is affected by distance from radio units, physical obstacles (walls, equipment), RF interference, and the number of simultaneously connected devices. Do not quote theoretical maximum speeds as expected performance.

## Guardrail 8: VPC Integration Uses Elastic Network Interfaces
Private 5G traffic routes through your VPC via elastic network interfaces (ENIs) in your specified subnet. The subnet must have available IP addresses, and security groups must allow the required traffic. Do not confuse this with VPN or Direct Connect.

## Guardrail 9: Network Sites Map to Physical Locations
Each network site represents a physical location with radio units. A network can have multiple sites. Do not confuse network sites with Availability Zones or AWS Regions — they are physical deployment locations.

## Guardrail 10: Orders Track Hardware Delivery Status
The orders API tracks hardware shipment status. Order status progresses through ACKNOWLEDGING, PREPARING, SHIPPING, DELIVERED. Do not assume hardware issues before checking order status.

## Guardrail 11: Device Identifiers Are Tied to SIM Cards
Device identifiers (IMSI, ICCID) are associated with AWS-provided SIM cards. Each SIM has a unique IMSI for network authentication. Do not attempt to manually assign or modify IMSI values.

## Guardrail 12: Network Deletion Requires Hardware Return
Deleting a Private 5G network may require returning hardware to AWS. Do not suggest network deletion as a quick troubleshooting step. Verify hardware return requirements before proceeding with deletion.
