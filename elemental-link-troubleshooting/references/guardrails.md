# AWS Elemental Link Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Elemental Link issue.

## Guardrail 1: Elemental Link Is Physical Hardware
Elemental Link is a physical hardware encoder that must be shipped to your location. It is NOT a software service or virtual appliance. Troubleshooting may require physical access to the device for cable checks, LED status inspection, and network port verification.

## Guardrail 2: Device Registration Is Automatic
When an Elemental Link device connects to the internet for the first time, it automatically registers with the AWS account associated with the device purchase. There is no manual registration API. If registration fails, the issue is network connectivity or account association.

## Guardrail 3: Two Hardware Models with Different Capabilities
Elemental Link HD supports 1080p encoding with HDMI input. Elemental Link UHD supports 4K/UHD encoding with HDMI and SDI inputs. Do not suggest SDI troubleshooting for Link HD devices or assume UHD capabilities on HD hardware.

## Guardrail 4: Firmware Is AWS-Managed
Firmware updates are pushed by AWS and cannot be manually installed. Operators can approve or defer updates but cannot upload custom firmware. Never suggest downloading firmware from external sources or manual firmware installation procedures.

## Guardrail 5: Bandwidth Requirements Are Critical
AWS recommends upload bandwidth of at least 2x the target encoding bitrate for reliable streaming. A 5 Mbps stream requires at least 10 Mbps upload. Insufficient bandwidth causes frame drops, quality degradation, and disconnections — not error messages.

## Guardrail 6: Devices Are Region-Specific
An Elemental Link device is registered to a specific AWS region. It can only be used with MediaLive channels in that same region. To use a device in a different region, it must be transferred using the transfer-input-device API. Transfers change the device's region association.

## Guardrail 7: Connection State vs Settings Sync State
ConnectionState indicates whether the device is currently communicating with AWS (CONNECTED/DISCONNECTED). DeviceSettingsSyncState indicates whether device settings match cloud configuration (SYNCED/SYNCING). These are independent — a device can be CONNECTED but not SYNCED.

## Guardrail 8: Input Device vs MediaLive Input Are Different Resources
An Elemental Link device (input device) and a MediaLive input are separate AWS resources. The device provides the video source; the input is the MediaLive resource that receives the video. They must be linked but are managed independently.

## Guardrail 9: Thumbnail API Shows Current Frame Only
The describe-input-device-thumbnail API returns a single JPEG frame of the current input. It does NOT provide video playback, recording, or historical frames. Use it to verify the device is receiving video signal, not for quality assessment.

## Guardrail 10: Device Transfer Changes Ownership
Transferring an Elemental Link device to another AWS account is a permanent ownership change until transferred back. The source account loses all access to the device. Pending transfers can be cancelled, but completed transfers cannot be reversed without the new owner's cooperation.

## Guardrail 11: Network Requirements Include Specific Ports
Elemental Link requires outbound HTTPS (443) and specific media ports to communicate with AWS. Corporate firewalls and proxy servers can block device connectivity even when general internet access works. Check firewall rules for AWS IP ranges.

## Guardrail 12: MediaLive Channel Must Be Stopped to Change Input
You cannot change the input device attachment on a running MediaLive channel. The channel must be stopped before modifying input configurations. Stopping a channel interrupts the live stream — coordinate with production teams before making changes.
