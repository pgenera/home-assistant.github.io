---
title: APRS
description: Instructions on how to track APRS stations with Home Assistant.
ha_release: 0.95
ha_categories:
  - presence_location.trackers
ha_iot_class: Cloud Push
ha_config_flow: true
ha_codeowners:
  - '@PhilRW'
  - '@pgenera'
ha_domain: aprs
ha_platforms:
  - device_tracker
  - diagnostics
ha_integration_type: service
ha_quality_scale: legacy
---

The **APRS** [(Automatic Packet Reporting System)](https://en.wikipedia.org/wiki/Automatic_Packet_Reporting_System) {% term integration %} connects Home Assistant to [APRS-IS](https://www.aprs-is.net/), the internet backbone of the amateur radio APRS network, to follow the positions of APRS stations and objects, such as a car, a hiker or a radiosonde, as device trackers.

## Use cases

- Show friends, family or your own vehicle on the Home Assistant map when they carry an APRS tracker.
- Trigger automations when an APRS station arrives at or leaves a zone.

## Prerequisites

- Tracking stations works without an amateur radio license or passcode: Home Assistant can log in to APRS-IS receive-only.
- Use a login callsign or SSID that none of the stations you track, and none of your other APRS programs, use.

{% include integrations/config_flow.md %}

If you track stations with the APRS `device_tracker` platform in {% term "`configuration.yaml`" %}, Home Assistant imports it as an entry when it starts. Remove the YAML configuration afterwards; it is deprecated, and a repair explains anything that stops the import. The imported trackers are new entities: remove the APRS stations from `known_devices.yaml`, and update automations and dashboards that use the old device tracker entities.

{% configuration_basic %}
Login callsign:
  description: "The callsign, optionally with an SSID, that Home Assistant logs in to APRS-IS with. Don't use the callsign-SSID of a station you want to track: APRS-IS does not send a station's packets back to the connection that uses its callsign."
Passcode:
  description: The APRS-IS passcode for the login callsign. Leave it empty to log in receive-only, which is all tracking needs.
Server:
  description: The APRS-IS server to connect to. The default, `rotate.aprs2.net`, picks a server for you.
Tracked callsigns:
  description: The stations to track, such as `N0CALL-9`. End a callsign with `*` to track every SSID, such as `N0CALL*`.
{% endconfiguration_basic %}

{% include integrations/option_flow.md %}

The only option is **Tracked callsigns**.

To change the server or passcode, go to {% my integrations title="**Settings** > **Devices & services**" %}, select {% icon "mdi:dots-vertical" %} on the entry and select **Reconfigure**. Leave the passcode empty to keep the current one, or enter `-1` to switch a receiver to receive-only.

## Supported functionality

### Device trackers

A tracked station gets a device and a device tracker the first time it is heard, so you don't need to know every SSID in advance. A tracker shows the last position heard, and these attributes when the station reports them:

- **GPS accuracy**: derived from the position ambiguity the station reports.
- **Altitude**: in meters.
- **Course** and **Speed**.
- **Comment**: the station's comment text.

Weather stations are not tracked, since they don't move.

## Data updates

The integration uses a push connection: APRS-IS sends positions as stations report them, filtered to the tracked callsigns. If the connection drops, Home Assistant reconnects, and the trackers are unavailable until it does.

## Examples

### Notify when a tracked car gets home

```yaml
automation:
  - alias: "Car is home"
    triggers:
      - trigger: zone
        entity_id: device_tracker.n0call_9
        zone: zone.home
        event: enter
    actions:
      - action: notify.notify
        data:
          message: "N0CALL-9 is home."
```

## Known limitations

- Each login callsign-SSID can only be used by one entry.
- A station is only tracked once it reports a position after Home Assistant connects; earlier positions are not fetched.

## Troubleshooting

### The passcode is not valid

APRS-IS passcodes belong to the callsign without its SSID. Check that the passcode matches the callsign, or leave it empty to log in receive-only. If APRS-IS refuses a passcode that worked before, Home Assistant asks you to enter it again.

### A tracked station doesn't update

1. Check that the callsign is in **Tracked callsigns**, with `*` if it uses different SSIDs.
2. Check that the station isn't using your login callsign-SSID.
3. Check that it appears on [aprs.fi](https://aprs.fi/). If it doesn't, it isn't reaching APRS-IS.

## Removing the integration

This integration follows standard integration removal. No extra steps are required.

{% include integrations/remove_device_service.md %}

Removing a tracked station's device removes its tracker until the station is heard again.
