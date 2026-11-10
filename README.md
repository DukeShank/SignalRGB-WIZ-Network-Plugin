# SignalRGB WiZ Network Plugin — WiZ Neo support

A small personal extension of the SignalRGB WiZ Network Plugin that lets SignalRGB control a Philips WiZ Neo RGB bulb over the local network.

## What

- Discovers WiZ devices on the local network using UDP broadcast, validates their identity and creates one single-LED SignalRGB device per discovered RGB device.
- Sends SignalRGB RGB colour and brightness to each device using the WiZ local `setPilot` command. Each device is represented by one logical LED.
- Probes previously discovered devices directly and updates their IP address when a later scan finds them at a new address. No IP or MAC address is stored in the code.
- Optionally turns the bulb off, or sets a shutdown colour, when SignalRGB exits or the PC goes to sleep.
- The plugin's service panel lists each discovered device's model, ID (MAC), room ID, IP address, firmware version and status.

## Why

The existing plugin did not work with my WiZ Neo. This repository contains the fix I needed, shared as-is in case it is useful to someone else.

## Tested configuration

| Property | Tested value |
| --- | --- |
| Bulb | Philips WiZ Neo RGB bulb |
| Module name | `ESP25_SHRGB_01` |
| Bulb firmware | 1.38.0 |
| SignalRGB | 2.5.86 |
| Operating system | Windows 11 |
| Network | 2.4 GHz Wi-Fi, same network as the PC |
| Plugin version | 1.3.0 |

Other WiZ models, firmware versions, SignalRGB versions and network setups may behave differently.

## Repository layout

```text
.
├── images/
│   └── wiz-logo.png      Device image and service icon (1024×1024, transparent)
├── .gitattributes        Line-ending rules for Git
├── README.md
├── WIZ.js                Plugin: discovery service and device component
└── WIZ.qml               Service panel shown on SignalRGB's Third Party Services page
```

## Install

**Requirements:** SignalRGB with add-on support, and a WiZ RGB bulb on the same 2.4 GHz network as the PC, with local communication enabled in the WiZ app (Settings > Security). Local UDP traffic must be allowed on the network.

1. In SignalRGB, open the add-on interface and add this repository's URL: `https://github.com/DukeShank/SignalRGB-WIZ-Network-Plugin`.
2. Select the `main` branch.
3. Install the add-on and restart SignalRGB if prompted.
4. Open the WiZ service on the Third Party Services page and wait for the bulb to appear with status **Online**.
5. Turn on **Automatically Start Stream** in the bulb's device settings.

## How it works

**Discovery:** the plugin broadcasts a WiZ `registration` request to UDP port 38899, validates the responding device's MAC address, then requests its model and firmware information with `getSystemConfig`. Each scan also sends direct probes to the last known IP addresses of discovered devices. The first five scans run about 5 seconds apart; after that, scans repeat every 15 seconds while no devices are known, or every 60 seconds once at least one device has been found. This is how address changes and device availability are detected.

Only devices whose module name contains `RGB` are added to SignalRGB. Other WiZ devices, such as smart plugs and white-only bulbs, are listed in the service panel with the status **Not added (not an RGB device)**. A device that has not answered discovery for 3 minutes is shown as **Not responding to discovery**.

**Control:** the device component sends RGB colour and dimming with `setPilot`. **Max Updates Per Second** limits how often frames are sent, checked on SignalRGB's render calls (about 30 per second). A value of 30 removes the limit, so changed frames can be sent on every render call. Unchanged frames are suppressed, then re-sent every 2 seconds in case a UDP packet was lost. Brightness is clamped to 10–100%. An incoming black frame uses the configured dimmed colour and minimum brightness instead.

If the control socket cannot be opened, the plugin retries every 2 seconds.

**Shutdown:** depending on **On Exit or Sleep**, the plugin sends nothing, a `setState` off command, or the **Shutdown Color** with `setPilot`. With **Shutdown Color**, the bulb is turned off instead when the PC goes to sleep.

## Settings

All settings are per device, in the bulb's device settings in SignalRGB.

| Setting | Default | Range / values | Description |
| --- | --- | --- | --- |
| Automatically Start Stream | Off | On / Off | Enables continuous frame streaming. |
| Max Updates Per Second | 10 | 1–30 | Limits the frame transmission rate. 30 means no limit. |
| Force Color | Off | On / Off | Uses **Forced Color** instead of the colour supplied by SignalRGB. |
| Forced Color | `#009bde` | any colour | Colour used when **Force Color** is on. |
| Default Color when dimmed | `#010101` | any colour | Colour used when the incoming SignalRGB frame is black. |
| Minimal Brightness | 10 | 10–100 % | Brightness used with the dimmed colour. |
| On Exit or Sleep | Leave as is | Leave as is / Turn off / Shutdown Color | What the plugin sends when SignalRGB exits or the PC goes to sleep. |
| Shutdown Color | `#ffb46b` | any colour | Colour sent at 100% brightness when SignalRGB exits, if **On Exit or Sleep** is **Shutdown Color**. |

## Status

**Verified with 1.3.0** on the tested configuration above:
- Discovery, device creation, RGB colour, dimming and continuous updates.
- Service panel: device details and the status line (**Online**).
- Discovery start is logged once instead of on every scan.
- Minimal Brightness shows as a number slider and keeps its saved value.
- On Exit or Sleep: **Leave as is**, **Turn off** and **Shutdown Color** on exit, and **Shutdown Color** on sleep (bulb off). After being turned off, the bulb comes back on when SignalRGB starts again.
- The `DeviceType(): property isn't callable` message logged with 1.2.3 no longer appears.

**Not yet verified on hardware:**
- Selecting purple and pink hues in Forced Color.
- The **Not responding to discovery** status after the bulb loses power.
- Control socket retry (tested only in a simulated run).
- The new logo, which SignalRGB loads from the `main` branch.

**Known limitations**
- Only RGB colour and dimming are sent. White temperature, scenes and groups are not supported.
- Discovery uses UDP broadcast plus direct probes of known IP addresses. It may fail on networks with client isolation or VLANs, or on PCs with several network adapters. There is no manual IP entry.
- A device that stops answering discovery is shown as not responding, but it is not removed automatically.
- Multiple bulbs, and device removal or reconnection, have not been tested.
- Shutdown commands are single UDP packets without confirmation; if a packet is lost, the bulb keeps its last colour.

## Troubleshooting

### The bulb does not appear in the service panel

- Confirm the bulb and the PC are on the same network, and that the bulb is on 2.4 GHz Wi-Fi.
- In the WiZ app, open Settings > Security and check that local communication is enabled.
- Guest networks, client (AP) isolation and VLANs can block broadcast discovery. Put the bulb and the PC on the same main network.
- On PCs with several network adapters (VPN, virtual machine or second Ethernet adapters), the broadcast may leave through the wrong adapter. Disconnect or disable the other adapters and restart SignalRGB.
- If Windows Firewall asks about SignalRGB, allow it on private networks, and check that the Wi-Fi or Ethernet network profile is set to Private.

### The bulb is listed as "Not added (not an RGB device)"

The plugin only adds devices whose module name contains `RGB`. Smart plugs and white-only bulbs are not supported.

### The bulb is listed but does not change colour

- Turn on **Automatically Start Stream** in the bulb's device settings.
- In the device's console in SignalRGB, check for `[WIZ] Bound device control socket to local port …` and `[WIZ] Device acknowledged live setPilot updates.`

### The bulb is listed as "Not responding to discovery"

The bulb has not answered for 3 minutes, for example because it was switched off at the wall or changed IP address. The plugin keeps probing; the status returns to **Online** when the bulb answers again.

## Disclaimer

This is a personal project with a fixed scope, not a maintained product.

- No support, updates, or compatibility guarantees for future SignalRGB, Windows, or WiZ firmware versions. Issues and pull requests may go unanswered.
- Only the device above has been tested. Other WiZ models may not work.
- Use at your own risk. You are responsible for testing any changes you make.
- WiZ is a trademark of Signify Holding. This project is not affiliated with or endorsed by Signify or SignalRGB. The logo is used only to identify the device.
- Keep traffic on your local network. Do not port-forward the bulb to the internet.
- Do not post your MAC addresses, IP addresses, or home/room IDs in issues or screenshots.

## Licence and attribution

The original plugin, [GreenSky-Productions/SignalRGB-WIZ-Network-Plugin](https://github.com/GreenSky-Productions/SignalRGB-WIZ-Network-Plugin) by GreenSky Productions (Philipp-Martin Seelk), does not include a licence file, and this fork does not add one. Without a licence, the original author keeps all rights to the original code; GitHub's terms of service allow viewing and forking it on GitHub. If the original author adds a licence, this section will be updated.

Credits for the plugin structure (SignalRGB Govee plugin) and the WiZ protocol details (openHAB WiZ binding) are listed in the header of `WIZ.js`.
