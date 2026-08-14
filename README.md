# Victron MultiPlus-II 3-input / shared 3-fan controller

ESPHome controller for up to three Victron MultiPlus-II units using one
**Seeed Studio XIAO ESP32-C6** and three 24 V 4-wire Noctua fans.

The controller reads the three original MultiPlus fan requests independently,
selects the highest request, quantizes it to 5% steps, then rescales that
request onto a configurable baseline-to-100% fan-speed range. One shared speed
command is sent to all three fans.

All three fans therefore run at the same commanded speed. Each fan keeps its
own tachometer channel so failures can still be detected independently.

## Hardware overview

![Schematic](schematic.svg)

## Home Assistant entities

**Requests**
- `MP1 Victron Fan Request`
- `MP2 Victron Fan Request`
- `MP3 Victron Fan Request`
- `Highest Victron Fan Request`

**Control**
- `Fan Mode` — Auto / Manual / Full Speed
- `Fan Speed` — live shared command and manual slider
- `Baseline Fan Speed` — Auto-mode speed at a 0% Victron request

Moving `Fan Speed` from Home Assistant automatically changes `Fan Mode` to
Manual. Select Auto to return control to the MultiPlus requests.

## Auto control logic

The three Victron request inputs are rounded to the nearest **5%** and the
highest request is used. In Auto mode that request is rescaled from
**0–100%** onto **Baseline Fan Speed–100%**:

`command = baseline + (request / 100) × (100 - baseline)`

The final fan command is rounded to the nearest whole percent. For example, a
50% baseline and a 50% Victron request produce a 75% fan command. A 0% request
runs at the baseline; a 100% request always commands 100%.

**Feedback**
- `MP1 Fan RPM`
- `MP2 Fan RPM`
- `MP3 Fan RPM`
- `MP1 Fan Fault`
- `MP2 Fan Fault`
- `MP3 Fan Fault`

## Pin allocation

| Function | XIAO pin | ESP32-C6 GPIO |
|---|---|---:|
| MP1 request | D1 | GPIO1 |
| MP2 request | D2 | GPIO2 |
| MP3 request | D3 | GPIO21 |
| **Shared 25 kHz fan PWM** | **D4** | **GPIO22** |
| MP1 tach | D5 | GPIO23 |
| MP2 tach | D7 / RX | GPIO17 |
| MP3 tach | D10 | GPIO18 |
| unused UART TX | D6 / TX | GPIO16 |

The selected control pins avoid the ESP32-C6 strapping pins. D6 / GPIO16 is
left unused so the shared fan PWM output is not placed on the XIAO UART-TX pin.

## ESPHome package structure

The local device YAML remains intentionally small and imports the controller
from this repository:

```yaml
packages:
  remote_package:
    url: https://github.com/Nxt-1/ESPHome-VictronMP-II-FanController
    ref: main
    files:
      - packages/controller.yaml
    refresh: 0s
```

The actual controller configuration is contained in
`packages/controller.yaml`.

## Victron request inputs

Each original two-wire fan output is read through its own PC817. The three
MultiPlus fan circuits therefore remain isolated from the ESP32 and from each
other.

## Fan PWM output

One 25 kHz LEDC output on D4 / GPIO22 drives three separate BC547B stages.
Each fan gets its own transistor, but all three transistors receive the same
PWM command.

With the XIAO unpowered or the PWM GPIO high-impedance, the BC547s are off and
the Noctua PWM inputs are released. The fans therefore default to full speed
as long as the 24 V fan supply remains present.

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
