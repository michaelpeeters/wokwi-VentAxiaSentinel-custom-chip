# Getting a Vent-Axia Sentinel Kinetic into Home Assistant — research notes

This repo simulates the UART "wired remote" protocol of a Vent-Axia Sentinel
Kinetic MVHR unit (see `src/main.c`) so that firmware talking to the real
unit can be developed/tested in Wokwi. This note tracks the known routes for
getting that data (and control) into Home Assistant, so we can pick the right
one to build/test against.

Which route applies depends on what interface your physical unit exposes:

| Your unit has... | Route | Status |
| --- | --- | --- |
| Only the wired remote / display port (RS232, 9600 8N1) — Kinetic | ESPHome external component | Community project, active development |
| A WiFi module (Sentinel Kinetic Advance S) | Local-network WiFi API | Existing HACS custom integration |
| A BMS terminal (RS485/Modbus) — Kinetic Advance / ComAir HRUC-Plus 3 | Modbus RTU over TCP gateway | Existing HACS custom integration, most mature/complete |
| Sentinel **Econiq** (Apex platform, RS485 or 868MHz RF) | Native Modbus RTU over RS485 + HA's built-in Modbus integration | Community guide, DIY register map from Vent-Axia support |
| Sentinel **Econiq** with WiFi (Vent-Axia Connect app) | Local-only WiFi API (undocumented) | Unconfirmed — under investigation, see section 5 |

Note: **Econiq is a different, newer product line from Kinetic** — it does
not use the wired-remote protocol this repo simulates, so routes 1 and the
Kinetic-specific Modbus repo below don't apply to it. See section 4.

## 1. Wired remote / UART route (matches this simulator)

This is the protocol this repo emulates: the unit drives its wired display
remote over RS232 at **9600 baud, 8N1**, sending a 42-byte packet roughly
every 300ms.

- **Voltage levels are RS232 (~±8.8V), not TTL/3.3V.** A level shifter
  (e.g. MAX3232) is required between the unit and any ESP32/Arduino — do not
  wire it directly to a microcontroller UART pin.
- Packet layout: 6-byte header (`02 00 00 08 07 15` for the normal display
  page, `02 08 46 89 00 15` for diagnostic pages), 16 ASCII bytes for line 1,
  a `16` separator, 16 ASCII bytes for line 2, then a 2-byte CRC. This lines
  up with the frames hard-coded in `src/main.c` (`Normal Airflow`,
  `Diagnostic 00`…`28`, etc).
- CRC is computed by starting at `0xFFFF` and subtracting each preceding
  byte's value — confirmed independently by the reverse-engineering repo
  below and by the CRC bytes already present in `src/main.c`.
- The same port also accepts 8-byte keyboard packets (`04 05 AF EF FB` +
  keycode + CRC) to inject Main/Up/Down/Set button presses, i.e. control as
  well as read-only monitoring is possible.

References:
- [aelias-eu/vent-axia-remote](https://github.com/aelias-eu/vent-axia-remote) — the reverse-engineering source for the above (oscilloscope-derived UART settings, packet/CRC breakdown, PoC Arduino code).
- [ESPHome external component thread](https://community.home-assistant.io/t/vent-axia-sentinel-kinetic-mvhr-external-component/900588) — an ESPHome external component built on this protocol; works with any unit that supports the Sentinel Kinetic wired remote. Exposes the display text, temperature/humidity, and diagnostic values as HA entities, plus button entities to drive Main/Up/Down/Set.
- [Vent-Axia MVHR Sentinel Kinetic serial port integration thread](https://community.home-assistant.io/t/vent-axia-mvhr-sentinel-kinetic-serial-port-intergration/675567) — earlier community thread wiring up the same serial port.

**This is the route this repo's simulator directly supports** — if we build
firmware against the custom chip here, it should be validated against (or
merged/contributed to) the ESPHome external component above rather than
reinventing the decoding.

## 2. WiFi module route

If the unit has the optional Sentinel Kinetic WiFi module fitted, no serial
wiring is needed at all — a Home Assistant custom integration talks to it
directly over the local network.

- [JosyBan/ventaxia_ha](https://github.com/JosyBan/ventaxia_ha) — HACS custom integration for the "Sentinel Kinetic Advance S" with WiFi module. Uses the `ventaxiaiot` Python library, needs the device IP, WiFi key, and device ID (printed on the unit). Exposes sensors, airflow-control buttons, and a service to set airflow. Actively maintained (v0.1.12, ~67 commits as of writing).

## 3. Modbus/BMS route

Units with a BMS terminal block (RS485) can be bridged to Home Assistant via
a Modbus RTU↔TCP gateway — no microcontroller/firmware needed at all.

- [Koky05/comair-modbus-homeassistant](https://github.com/Koky05/comair-modbus-homeassistant) — HACS integration for ComAir HRUC-Plus 3 / Vent-Axia Sentinel **Kinetic Advance** MVHR via Modbus RTU over TCP. Uses an Elfin EW11A RS485→TCP gateway wired to the unit's BMS connector (RJ12, powered off the unit itself). Exposes ~38 entities: temps, humidity, CO2, fan RPM, power/energy, a full `climate` entity with presets, mode `select`, and BMS override `switch`es. This is the most feature-complete of the three Kinetic options, but only applies to Kinetic Advance units with the BMS interface enabled/wired out — it does not target Econiq.

## 4. Sentinel Econiq route (separate product line, not this repo's protocol)

The **Econiq** (built on Vent-Axia's newer "Apex" platform) is a different
unit from the Kinetic this repo simulates. It doesn't speak the wired-remote
display protocol at all — instead it exposes **native Modbus RTU** over its
BMS RS485 terminal (or an optional 868MHz RF link), so there's no custom
firmware/decoding to write.

- Enable RS485/Modbus comms mode and set the unit's Modbus address first via
  the **Vent-Axia Connect** app.
- Wire a standard RS485↔TCP or RS485↔USB gateway to the BMS terminal, then
  use Home Assistant's built-in **Modbus integration** (`modbus:` in
  `configuration.yaml`) directly — no custom component needed.
- The register map and comms parameters (baud rate, parity, holding/input
  register addresses) are **not publicly published** by Vent-Axia; the
  approach used in the community guide was to request them directly from
  **Vent-Axia technical support**.
- [Vent-Axia Sentinel Econiq – Modbus/RS485 integration guide](https://community.home-assistant.io/t/vent-axia-sentinel-econiq-modbus-rs485-integration/993007) — the community write-up of the above (HA Community "Community Guides" section; blocked from automated fetch by Cloudflare, but summarized here from search indexing — worth reading directly for the full register list).

## 5. Econiq WiFi route (investigation in progress)

Confirmed on our own unit: the Econiq's Vent-Axia Connect app can talk to it
over **WiFi**, and this only works while the phone is on the same home WiFi
— it stops working over a VPN (tested with Tailscale). That behaviour is a
useful data point, not just an inconvenience:

- **It means there's no cloud relay** — the app is doing local device
  control, the same architecture as the Kinetic Advance S WiFi module
  (`ventaxiaiot`: manual IP address + WiFi key + Device ID, no cloud
  round-trip). The Econiq's WiFi and BLE both hang off the same Vent-Axia
  Connect app, so the underlying local API is plausibly similar or shared,
  but this is **not yet confirmed** — JosyBan/ventaxia_ha's docs only name
  "Sentinel Kinetic Advance S" explicitly, not Econiq.
- **Why the VPN breaks it**: the app almost certainly relies on local
  discovery (mDNS/UDP broadcast) to find the unit, or the VPN client is
  full-tunnelling all traffic off the LAN. Broadcast/multicast fundamentally
  cannot cross a Layer-3 point-to-point overlay like Tailscale/WireGuard —
  there's no shared broadcast domain to send it on. This is a long-standing,
  still-open Tailscale feature request
  ([tailscale/tailscale#1013](https://github.com/tailscale/tailscale/issues/1013),
  [#11134](https://github.com/tailscale/tailscale/issues/11134)), not a bug
  or a deliberate block, and it isn't specific to Vent-Axia — any mDNS-based
  discovery protocol will fail the same way over Tailscale.
- **What still works over Tailscale**: plain unicast IP. A
  [subnet router](https://tailscale.com/docs/features/subnet-routers)
  advertising the home LAN gets you a routable path to the unit's specific
  IP:port — only the *discovery* step is broken, not connectivity once the
  IP is known.

### Next steps to pin down the protocol

1. **Find the unit's LAN IP once** (router DHCP leases, or the Connect app's
   device info screen) and see if the app/API works when addressed directly
   by IP instead of via discovery — mirrors how `ventaxiaiot` already avoids
   discovery entirely by requiring a manual IP.
2. **Passive packet capture** of the Connect app talking to the unit while
   both are on the home WiFi (Wireshark on a mirrored/monitor port, or
   PCAPdroid on Android — no root needed) to identify the transport
   (HTTP/UDP/TCP), port, and payload format. Zero risk to the unit — this is
   just sniffing our own WiFi traffic, not touching the device.
3. If the payload structure matches `ventaxiaiot`'s scheme, try pointing
   JosyBan/ventaxia_ha at the Econiq's IP/key/device-ID directly, or open an
   issue/PR asking about Econiq support.
4. If mDNS discovery turns out to be load-bearing and can't be dropped,
   workarounds (roughly in order of how much complexity they add) are: an
   mDNS reflector (`avahi-daemon` reflector mode / `mdns-repeater`) running
   on a LAN box that's also in the tailnet, or swapping Tailscale for
   ZeroTier for this link specifically (ZeroTier is Layer 2 and does forward
   multicast/mDNS/broadcast, unlike Tailscale).

### Practical implication

Once the Econiq's data is flowing into Home Assistant (which lives
permanently on the home LAN), the "must be on the same WiFi" constraint
stops being a problem for *us* — HA does the local discovery/connection
once, and we access HA itself remotely (Tailscale, Nabu Casa, etc.) over an
ordinary unicast HTTPS connection. HA becomes the bridge between "local-only
device" and "reachable from anywhere," which is the actual goal here.

## 6. Other leads for the Econiq WiFi investigation

Beyond straight packet capture (section 5), a few more angles to try, roughly
cheapest/fastest first:

- **Check whether the Econiq is a rebrand.** Confirmed precedent: Vent-Axia's
  own bathroom extractor fans are OEM'd from the Swedish brand Pax —
  [`eriknn/ha-pax_ble`](https://github.com/eriknn/ha-pax_ble) explicitly
  documents "Vent-Axia Svara (same as the [Pax] Calima)" and "Vent-Axia
  Svensa (same as [Pax] PureAir Sense)," same BLE protocol and all. Vent-Axia
  is known to relabel other manufacturers' hardware for at least part of its
  range, and its Dutch site calls the Econiq a generic "WTW-unit" (the
  catch-all Dutch/Benelux term for MVHR). Worth checking a CE/compliance
  label inside the unit or on the WiFi module for the real manufacturer name
  — if it's a rebrand, a reverse-engineered protocol or HA integration may
  already exist under a different brand name.
- **Check whether the WiFi module is a Tuya-style module.** Many white-label
  WiFi appliance modules in the EU are Tuya IoT modules with a documented
  local-key protocol, for which a mature zero-cloud HACS integration already
  exists (**LocalTuya**). Tell-tale sign: a "SmartLife"-style QR-code/AP-mode
  pairing flow in the Connect app. If it matches, extracting the device's
  `local_key` via the Tuya IoT developer console could be the fastest path
  to a working integration — no protocol reverse-engineering needed at all.
- **Decompile the Vent-Axia Connect APK** (`jadx`/`apktool`) and grep the
  decompiled source/strings for URLs, ports, and SDK names (e.g. "tuya",
  "esp32", vendor SDK identifiers) before doing any live packet capture —
  static analysis is faster than sniffing and narrows down what to look for.
- **Check the Play Store listing's declared permissions** for a quick, no-
  decompile signal: "Nearby devices"/local network access and/or Bluetooth
  and precise-location permissions indicate WiFi-local vs. BLE vs. both.
- **Post findings to the community.** All existing Econiq HA discussion
  covers the RS485/Modbus route only — nobody's publicly tackled the
  WiFi/app side yet. Sharing a packet capture or APK findings on the HA
  Community (e.g. as a follow-up on the
  [Econiq Modbus/RS485 thread](https://community.home-assistant.io/t/vent-axia-sentinel-econiq-modbus-rs485-integration/993007))
  is how the Kinetic wired-remote and Pax/Svara BLE protocols got cracked in
  the first place — community reverse-engineering, not a lone effort.

## Recommendation

- If the goal is to keep using/extending **this repo's simulator** (Kinetic
  wired-remote protocol), target the ESPHome external component (route 1) —
  same protocol, same packets.
- If the physical unit already has a WiFi module or Kinetic Advance BMS/Modbus
  port fitted, routes 2 or 3 are turnkey (no custom firmware/wiring) and more
  mature — prefer them over DIY UART sniffing.
- If the unit is actually a **Sentinel Econiq**, this repo's simulator doesn't
  apply. Two live options: request the register map from Vent-Axia and use
  HA's stock Modbus integration over RS485 (route 4, works today); or, since
  our unit already has WiFi via the Connect app, investigate the local WiFi
  API per sections 5–6 — no wiring/case-opening required, but the protocol
  isn't confirmed yet. Cheapest first steps: check the Play Store permissions
  and whether pairing looks Tuya-like, before reaching for Wireshark.
