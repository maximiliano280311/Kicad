# Rocket Flight Computer — KiCad Project

## Files in this package

| File | Purpose |
|------|---------|
| `rocket_fc.kicad_pro` | KiCad project file (opens the whole project) |
| `rocket_fc.kicad_sch` | KiCad 7/8 schematic (S-expression format) |
| `rocket_fc.net` | KiCad netlist (legacy export format, importable into PCB editor) |
| `rocket_fc_bom.csv` | Bill of Materials with MPNs and footprints |

---

## How to open in KiCad 7 or 8

1. Open KiCad
2. File → Open Project → select `rocket_fc.kicad_pro`
3. Double-click `rocket_fc.kicad_sch` to open the schematic editor
4. To start PCB layout: Tools → Update PCB from Schematic (or import `rocket_fc.net` manually)

---

## Required KiCad Libraries

All footprints and symbols reference standard KiCad built-in libraries:

- `Device` — resistors, capacitors, inductors, generic discretes
- `Connector` / `Connector_PinHeader_2.54mm` — headers and screw terminals
- `Connector_JST` — battery JST-XH connector
- `Connector_Phoenix_MC_HighVoltage` — pyro screw terminals
- `Connector_Card` — microSD socket
- `Package_TO_SOT_SMD` / `Package_TO_SOT_THT` — transistors and MOSFETs
- `Package_SO` — SOIC-8 (MP1584)
- `Package_LGA` — sensor ICs (MPU-6050, BMP280)
- `Package_LCC` — MS5611
- `LED_SMD` — status LEDs
- `Diode_SMD` — flyback diode
- `Buzzer_Beeper` — piezo buzzer
- `Button_Switch_SMD` / `Button_Switch_THT` — tactile and arming switches
- `RF_Module` — ESP32-WROOM-32D, SX1278/RFM95W
- `RF_GPS` — u-blox NEO-8M
- `Regulator_Linear` — AMS1117-3.3
- `Regulator_Switching` — MP1584 (may need custom symbol; see note below)
- `Inductor_SMD` — Bourns SRR1260 inductor

### Missing symbol note

If KiCad cannot find `Regulator_Switching:MP1584`, create a custom symbol with these pins:
- Pin 1: VIN (power_in)
- Pin 2: GND (power_in)
- Pin 3: EN (input)
- Pin 4: SS (input, tie to GND via 100nF)
- Pin 5: FB (input)
- Pin 6: COMP (input, tie via RC to GND)
- Pin 7: BST (passive, 100nF to SW)
- Pin 8: SW (output)

---

## PCB Net Classes (from .kicad_pro)

| Net Class | Track Width | Clearance | Nets |
|-----------|------------|-----------|------|
| Default | 0.25 mm | 0.2 mm | All signal traces |
| Power_3V3 | 1.0 mm | 0.25 mm | +3.3V |
| Power_5V | 1.5 mm | 0.25 mm | +5V, +5V_SERVO |
| Pyro_High_Current | **2.5 mm** | **1.27 mm** | PYRO_BUS_P, PYRO_A/B_DRAIN, VBAT_* |
| SPI_Bus | 0.3 mm | 0.2 mm | SPI_SCK, MOSI, MISO, CS lines |
| I2C_Bus | 0.25 mm | 0.2 mm | I2C_SDA, I2C_SCL |

---

## Critical Design Rules to Verify in DRC

After importing netlist and placing components, run DRC and check:

1. **Pyro clearance**: All PYRO_HIGH_CURRENT nets must maintain 1.27 mm clearance from logic nets
2. **ESP32 antenna keepout**: No copper within 15 mm below the WROOM module's antenna end
3. **GPS keepout**: No copper fill under NEO-8M patch antenna area
4. **Decoupling placement**: C7, C9, C10, C12, C14, C15 must be within 0.5 mm of their IC power pins
5. **C24 servo bulk cap**: Must be placed within 5 mm of the servo header power rail

---

## Safety-Critical Components

The following components must NEVER be substituted or omitted:

| Ref | Value | Why critical |
|-----|-------|-------------|
| R7 | 10k pull-down | Holds Q2 gate LOW — prevents pyro Ch A firing at power-on |
| R8 | 10k pull-down | Holds Q3 gate LOW — prevents pyro Ch B firing at power-on |
| SW1 | Arming switch | Hard disconnect — no software path bypasses this |
| R5, R6 | 100R gate | Protect ESP32 GPIO from MOSFET Ciss inrush spike |
| C23 | 100nF EN RC | Prevents ESP32 boot-loop from power supply glitch |

---

## Boot Pin Constraints (must be verified before power-up)

| GPIO | At Boot | Requirement | Circuit |
|------|---------|-------------|---------|
| GPIO0 | HIGH | Normal boot | R4 10k pull-up to 3.3V |
| GPIO2 | LOW or HIGH | Both OK | LED/LoRa RESET use |
| GPIO5 | HIGH | Boot OK | R22 100k pull-up |
| GPIO12 | LOW | Required | LED3 keeps it LOW |
| GPIO15 | HIGH | Silences boot messages | R21 100k pull-up |

---

## Firmware Notes

- Use ESP32 LEDC peripheral for servo PWM (hardware, not software) to avoid jitter
- Never assert both LORA_CS (GPIO5) and SD_CS (GPIO15) simultaneously
- GPIO2 is shared between LoRa RESET and GPS status LED — manage in firmware
- GPIO6-11 are bonded to internal WROOM flash — these pins do not appear in schematic and must never be used

---

## Gerber Export Settings (for JLCPCB / PCBWay)

- Copper layers: F.Cu, B.Cu, In1.Cu (GND), In2.Cu (Power)
- Board outline: Edge.Cuts
- Drill: Excellon format, separate PTH/NPTH
- File format: RS-274X
- Resolution: 4:6
- Silkscreen: F.Silkscreen, B.Silkscreen
- Solder mask: F.Mask, B.Mask
- Paste: F.Paste (no B.Paste for this design)
