# AERIS-10 FPGA Constraint Files

## Two Targets

| File | Device | Package | Purpose |
|------|--------|---------|---------|
| `xc7a50t_ftg256.xdc` | XC7A50T-2FTG256I | FTG256 (256-ball BGA) | Upstream author's board (copy of `cntrt.xdc`) |
| `xc7a200t_fbg484.xdc` | XC7A200T-2FBG484I | FBG484 (484-ball BGA) | Production board (new PCB design) |

## Why Two Files

The upstream prototype uses a smaller XC7A50T in an FTG256 package. The production
AERIS-10 radar migrates to the XC7A200T for more logic, BRAM, and DSP resources.
The two devices have completely different packages and pin names, so each needs its
own constraint file. Both files constrain the same RTL top module (`radar_system_top.v`).

## Bank Voltage Assignments

### XC7A50T-FTG256 (Upstream)

| Bank | VCCO | Signals |
|------|------|---------|
| 0 | 3.3V | JTAG, flash CS |
| 14 | 3.3V | ADC LVDS (LVDS_33), SPI flash |
| 15 | 3.3V | DAC, clocks, STM32 3.3V SPI, DIG bus |
| 34 | 1.8V | ADAR1000 control, SPI 1.8V side |
| 35 | 3.3V | Unused (no signal connections) |

### XC7A200T-FBG484 (Production)

| Bank | VCCO | Used/Avail | Signals |
|------|------|------------|---------|
| 13 | 3.3V | 17/35 | Debug overflow (doppler bins, range bins, status) |
| 14 | 2.5V | 19/50 | ADC LVDS_25 + DIFF_TERM, ADC power-down |
| 15 | 3.3V | 27/50 | System clocks (100M, 120M), DAC, RF, STM32 3.3V SPI, DIG bus |
| 16 | 3.3V | 50/50 | FT601 USB 3.0 (32-bit data + byte enable + control) |
| 34 | 1.8V | 19/50 | ADAR1000 beamformer control, SPI 1.8V side |
| 35 | 3.3V | 50/50 | Status outputs (beam position, chirp, doppler data bus) |

## Signal Differences Between Targets

| Signal | Upstream (FTG256) | Production (FBG484) |
|--------|-------------------|---------------------|
| FT601 USB | Unwired (chip placed, no nets) | Fully wired, Bank 16 |
| `dac_clk` | Not connected (DAC clocked by AD9523 directly) | Routed, FPGA drives DAC |
| `ft601_be` width | `[1:0]` in upstream RTL | `[3:0]` (RTL updated) |
| ADC LVDS standard | LVDS_33 (3.3V bank) | LVDS_25 (2.5V bank, better quality) |
| Status/debug outputs | No physical pins (commented out) | All routed to Banks 35 + 13 |

## How to Select in Vivado

In the Vivado project, only one XDC should be active at a time:

1. Add both files to the project: `File > Add Sources > Add Constraints`
2. In the Sources panel, right-click the XDC you do NOT want and select
   `Set File Properties > Enabled = false` (or remove it from the active
   constraint set)
3. Alternatively, use two separate constraint sets and switch between them

For TCL-based flows:
```tcl
# For production target:
read_xdc constraints/xc7a200t_fbg484.xdc

# For upstream target:
read_xdc constraints/xc7a50t_ftg256.xdc
```

## Notes

- The production XDC pin assignments are **recommended** for the new PCB.
  The PCB designer should follow this allocation.
- Bank 16 (FT601) is fully utilized at 50/50 pins. No room for expansion
  on that bank.
- Bank 35 (status/debug) is also at capacity (50/50). Additional debug
  signals should use Bank 13 spare pins (18 remaining).
- Clock inputs are placed on MRCC (Multi-Region Clock Capable) pins to
  ensure proper clock tree access.
