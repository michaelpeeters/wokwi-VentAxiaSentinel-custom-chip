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

## Recommendation

- If the goal is to keep using/extending **this repo's simulator** (Kinetic
  wired-remote protocol), target the ESPHome external component (route 1) —
  same protocol, same packets.
- If the physical unit already has a WiFi module or Kinetic Advance BMS/Modbus
  port fitted, routes 2 or 3 are turnkey (no custom firmware/wiring) and more
  mature — prefer them over DIY UART sniffing.
- If the unit is actually a **Sentinel Econiq**, this repo's simulator doesn't
  apply — go straight to route 4 (request the register map from Vent-Axia,
  then use HA's stock Modbus integration).
