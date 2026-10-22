---
title: APRS
description: Instructions on how to track APRS stations and publish your own position and weather to APRS-IS with Home Assistant.
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
  - sensor
ha_integration_type: service
ha_quality_scale: legacy
---

The **APRS** [(Automatic Packet Reporting System)](https://en.wikipedia.org/wiki/Automatic_Packet_Reporting_System) {% term integration %} connects Home Assistant to [APRS-IS](https://www.aprs-is.net/), the internet backbone of the amateur radio APRS network. It can do two things, each set up as its own entry:

- **Track stations**: follow the positions of APRS stations and objects, such as a car, a hiker or a radiosonde, as device trackers.
- **Publish a station**: send the position of a zone, person or device tracker to APRS-IS, optionally with weather data from your sensors, so it shows up on maps such as [aprs.fi](https://aprs.fi/).

## Use cases

- Show friends, family or your own vehicle on the Home Assistant map when they carry an APRS tracker.
- Trigger automations when an APRS station arrives at or leaves a zone.
- Feed your home weather station to APRS-IS, where weather networks such as the [Citizen Weather Observer Program (CWOP)](http://www.wxqa.com/) can pick it up.
- Beacon your phone's position over APRS-IS while you travel, without a radio.

## Prerequisites

- You need an amateur radio license and callsign to publish a station, and to log in with a passcode.
- Publishing needs the APRS-IS passcode for your callsign. Several APRS programs and websites can generate it from your callsign. Tracking stations works without a passcode.
- Use a different callsign or SSID for each entry, and for each of your own radios or programs that connect to APRS-IS. For example, use `N0CALL-13` for a weather station and `N0CALL` to track stations.

{% include integrations/config_flow.md %}

When adding the integration, choose **Track stations** or **Publish a station**.

If you track stations with the APRS `device_tracker` platform in {% term "`configuration.yaml`" %}, Home Assistant imports it as a **Track stations** entry when it starts. Remove the YAML configuration afterwards; it is deprecated, and a repair explains anything that stops the import. The imported trackers are new entities: remove the APRS stations from `known_devices.yaml`, and update automations and dashboards that use the old device tracker entities.

### Track stations

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

### Publish a station

The first step logs in to APRS-IS:

{% configuration_basic %}
Callsign:
  description: The callsign and SSID the station reports as, such as `N0CALL-13`. Use an SSID that none of your other APRS stations use.
Passcode:
  description: The APRS-IS passcode for the callsign.
Server:
  description: The APRS-IS server to send reports to.
{% endconfiguration_basic %}

The second step chooses what the station reports:

{% configuration_basic %}
Location:
  description: The zone, person or device tracker whose position is reported. Use **Home** for a fixed station such as a weather station.
Symbol:
  description: "The APRS symbol table character (`/`, `\\`, or an overlay digit or capital letter) and symbol code, such as `/_` for a weather station, `/-` for a house or `/>` for a car. Weather data can only be sent with the weather station symbol, `_`."
Comment:
  description: Text sent at the end of each report. It can be up to 43 characters, or 36 for a weather station or a moving location. Use printable ASCII characters other than `|` and `~`. A weather station's comment can't start with a letter and a digit, such as `h50`, since that reads as weather data.
Position ambiguity:
  description: How many trailing digits of the position to leave out, from 0 (exact, about 18 m) to 4 (the nearest degree), to report the position less precisely.
Weather:
  description: The sensors to report, one for each weather field. Only sensors with a matching device class are offered. Leave a field empty to leave it out.
{% endconfiguration_basic %}

The weather fields are temperature, humidity, sea level pressure, wind direction, wind speed, wind gust, rain in the last hour, rain in the last 24 hours, rain since midnight, and solar irradiance. Values are converted to the units APRS uses, so the sensors can use any unit Home Assistant can convert.

{% include integrations/option_flow.md %}

The options are the same as when setting up the entry: the tracked callsigns for a receiver, or what a station reports.

To change the server or passcode, go to {% my integrations title="**Settings** > **Devices & services**" %}, select {% icon "mdi:dots-vertical" %} on the entry and select **Reconfigure**. Leave the passcode empty to keep the current one, or enter `-1` to switch a receiver to receive-only.

## Supported functionality

### Device trackers

A tracked station gets a device and a device tracker the first time it is heard, so you don't need to know every SSID in advance. A tracker shows the last position heard, and these attributes when the station reports them:

- **GPS accuracy**: derived from the position ambiguity the station reports.
- **Altitude**: in meters.
- **Course** and **Speed**.
- **Comment**: the station's comment text.

Weather stations are not tracked, since they don't move.

### Sensors

A published station has a **Last report** diagnostic sensor with the time of the last report that APRS-IS accepted. If it stops advancing, the station has stopped reporting.

## Data updates

Tracking uses a push connection: APRS-IS sends positions as stations report them, filtered to the tracked callsigns. If the connection drops, Home Assistant reconnects, and the trackers are unavailable until it does.

A published station connects only to send a report:

- With weather data, it reports every 10 minutes, as weather networks ask.
- A position-only report for a zone goes out every 30 minutes.
- A person or device tracker reports using SmartBeaconing™: every 30 minutes when stopped, more often the faster it moves, down to every 3 minutes at 60 mph (about 100 km/h) and above, and soon after it turns or stops. Moving reports include the course and speed. Position changes within the location's GPS accuracy are ignored.

SmartBeaconing™ is a trademark of HamHUD.net.

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

### Publish your phone's position while you travel

Add a station with your phone's person or device tracker as the **Location** and the `/>` car symbol. It reports more often while you drive, with your course and speed, and every 30 minutes while you are stopped.

## Known limitations

- A station logs in with its passcode but does not handle APRS messages, so its reports are marked as from a station without messaging.
- Reports are sent to APRS-IS only, not over radio.
- Each callsign-SSID can only be used by one entry, whether it tracks stations or publishes one.
- A station is only tracked once it reports a position after Home Assistant connects; earlier positions are not fetched.
- Positions are reported to a hundredth of a minute, about 18 m.
- Reports are identified as coming from Home Assistant by the experimental `APZHAS` destination.

## Troubleshooting

### The passcode is not valid

APRS-IS passcodes belong to the callsign without its SSID. Check that the passcode matches the callsign. If APRS-IS refuses a passcode that worked before, Home Assistant asks you to enter it again.

### A station doesn't appear on aprs.fi

1. Check that the station's **Last report** sensor is advancing.
2. Check that the location entity has a position, for example on the map.
3. Look for warnings from the APRS integration in the logs. A failed report is logged once, and again when reports resume.
4. Check the raw packets for your callsign on [aprs.fi](https://aprs.fi/) to see whether APRS-IS received them.

### A tracked station doesn't update

1. Check that the callsign is in **Tracked callsigns**, with `*` if it uses different SSIDs.
2. Check that the station isn't using your login callsign-SSID.
3. Check that it appears on [aprs.fi](https://aprs.fi/). If it doesn't, it isn't reaching APRS-IS.

## Removing the integration

This integration follows standard integration removal. No extra steps are required. A station stops reporting when its entry is removed.

{% include integrations/remove_device_service.md %}

Removing a tracked station's device removes its tracker until the station is heard again.
