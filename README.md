# Victron MultiPlus-II 3-input / shared 3-fan controller

ESPHome controller for up to three Victron MultiPlus-II units using one
**Seeed Studio XIAO ESP32-C3** and three 24 V 4-wire Noctua fans.

The controller reads the three original MultiPlus fan requests independently,
selects the highest request, applies a configurable minimum speed, then sends
one shared speed command to all three fans.

All three fans therefore run at the same commanded speed. Each fan keeps its
own tachometer channel so failures can still be detected independently.

## Hardware overview

![Schematic](hardware/schematic.svg)


## Home Assistant entities

**Requests**
- `MP1 Victron Fan Request`
- `MP2 Victron Fan Request`
- `MP3 Victron Fan Request`
- `Highest Victron Fan Request`

**Control**
- `Fan Mode` — Auto / Manual / Full Speed
- `Fan Speed` — live shared command and manual slider
- `Minimum Fan Speed` — Auto-mode floor

Moving `Fan Speed` from Home Assistant automatically changes `Fan Mode` to
Manual. Select Auto to return control to the MultiPlus requests.

**Feedback**
- `MP1 Fan RPM`
- `MP2 Fan RPM`
- `MP3 Fan RPM`
- `MP1 Fan Fault`
- `MP2 Fan Fault`
- `MP3 Fan Fault`

## Pin allocation

| Function | XIAO pin | GPIO |
|---|---|---:|
| MP1 request | D1 | GPIO3 |
| MP2 request | D2 | GPIO4 |
| MP3 request | D3 | GPIO5 |
| **Shared 25 kHz fan PWM** | **D4** | **GPIO6** |
| MP1 tach | D5 | GPIO7 |
| MP2 tach | D7 / RX | GPIO20 |
| MP3 tach | D10 | GPIO10 |
| unused | D4 | GPIO6 |
| unused strapping pin | D0 | GPIO2 |
| unused strapping pin | D8 | GPIO8 |
| BOOT / unused | D9 | GPIO9 |

## Victron request inputs

Each original two-wire fan output is read through its own PC817. The three
MultiPlus fan circuits therefore remain isolated from the ESP32 and from each
other.

## Power

The design assumes:
- one 24 V Mean Well supply for all three fans;
- one 5 V Mean Well supply for the XIAO;
- 24 V PSU ground, 5 V PSU ground, XIAO ground and fan grounds are common;
- Victron fan-output grounds remain isolated through the PC817s.

Three NF-F12 industrialPPC-24V-3000 SP IP67 PWM fans can draw 0.18 A each, so
a 24 V / 1 A supply gives comfortable margin.

If the auxiliary supplies are AC/DC, feed them from an inverter-backed AC
source that remains available whenever the MultiPlus system can be operating
and producing heat.

## Build order

1. Flash the XIAO and verify HA entities.
2. Build one BC547 fan-output stage and verify manual PWM.
3. Add one tach input and verify RPM/fault detection.
4. Build one PC817 request input and compare it with the Victron output on a scope.
5. Test reset, 5 V controller-power loss and Wi-Fi/HA loss.
6. Duplicate the proven circuits for channels 2 and 3.
7. Enable Auto mode and verify highest-request control.
