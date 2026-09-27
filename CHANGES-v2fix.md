# MiciMike v2.1 (v2-fix) — changes from the "v2 hardware" commit (7de6eec)

This branch starts from the batch-2 ("v2 hardware") design, **not** the later ESP32-C6 work-in-progress.
It applies the fixes from issue #53 and a few bugs found while checking the design.
The reference for each fix is the Home Assistant Voice PE v1.0: its KiCad files, its ESPHome config, and the `voice-kit-xmos-firmware` port map.

**Status: untested candidate.** It passes ERC and DRC (see "Known / accepted DRC items" below), but no one has built it.
Have someone who does PCB layout review it before ordering.

## Schematic / electrical fixes

| # | Problem in v2 | Fix |
|---|---|---|
| 1 | XMOS 0.9 V core fed from a 200 mA LDO (TPS7A0309, U6) off 3.3 V | U6 is now a **TPS62065DSGR** 2 A buck running from +5 V: 1 µH (L8, Murata DFE201610E-1R0M), 51k/100k divider + 22 pF feed-forward, 10 µF in and 2×10 µF out. This copies the Voice PE v1.0 circuit exactly (Vout = 0.6 × 1.51 = 0.906 V). |
| 2 | Core ferrite FB3 was a 600 Ω / 500 mA part, and there was little bulk capacitance on VDD | FB3 → BLM18PG121SN1D (120 Ω, 2 A, low DCR). C44 on VDD → 10 µF. |
| 3 | XMOS cut off from the speaker path: no echo-cancellation reference, and the pinout didn't match the stock XMOS firmware | Restored the Voice PE pin roles. **X1D01 → I2S_LRCK, X1D10 → I2S_BCLK, X1D34 → I2S_DIN_ESP**. The XMOS now clocks the ESP (secondary) and the TAS5805M and feeds the amp its data. The ESP sends playback to X1D00 (the AEC reference). X1D11 (MCLK out) is left unconnected in v2.1: neither the TAS5805M nor the ESP in secondary mode uses MCLK. |
| 4 | U12 channel 1 wired backwards (mic data on 1OE, MUTE_ON on 1A), and X1D11 reused for mic 3 | Channel 1 tied off like the Voice PE (1OE = 1A = GND, 1Y open). Mic 3 (MK3, R61, R64) marked DNP. The stock firmware uses the MK1/MK2 stereo pair on X1D13. |
| 5 | XMOS debug header J4 referenced 3.3 V and had +5 V on a pin; J3 had no GND | J3 removed. J4 → 2×5 1.27 mm header with the Voice PE / XTAG "xSYS2" pinout for pins 1–10 (1 = +1V8 VREF, 2 TMS, 4 TCK, 6 TDO, 8 TDI, 10 RST_N, odd = GND). J4 is **DNP**: fit it only to flash a blank XMOS. XLink pins (X1D16–19) are left unconnected, and R37/R38 are DNP. |
| 6 | **TAS5805M ADR/FAULT tied to ground through 0 Ω (R70).** Per the TAS5805M datasheet (Table 7-5), the address is set only by a pull-up to DVDD (4.7k/15k/47k/120k). Grounded is not a valid setting. | R70 → **15 kΩ to DVDD**, giving I²C address **0x2D** (the tas58xx ESPHome component's default). |
| 7 | SK6812 LEDs on 5 V driven directly from a 3.3 V GPIO | Added U16 **SN74LV1T34** (5 V supply, TTL-level input) between the ESP LED GPIO and LED1 DIN, plus C103 100 nF. |
| 8 | ESP GPIO17 (XU316_RST) cannot be escaped from its pad without routing under the ESP32 QFN; the old route did exactly that | v2.1: **XU316_RST moved to GPIO21** (pad 27) and the **LED data moved to GPIO4** (pad 9, previously unused). GPIO17 is now a no-connect. |

## v2.1 layout clean-up (routing review)

The first v2-fix routing was DRC-clean but untidy: 66 % of the new segments were at arbitrary angles, there were 10 mid-trace T-junctions, 33 pad entries were off-centre, and the new routes used 62 vias. All added routing was ripped up and redone:

* **Every new segment is 0°, 45° or 90°** (the original v2 board is also 100 % octilinear). The only exception is the pre-existing +3V3 bridge, which is now vertical too.
* **No T-splits.** Every branch now happens at a pad or at a via. Tracks enter pads along the pad axis and end at the pad centre.
* **JTAG is one parallel bundle.** RST_N, TDI, TDO, TMS, TCK and the +1V8 reference run at a constant 0.26 mm pitch from the XMOS to J4. They fan into the header with one TCK layer hop.
  * J4 now sits on the top side, in the same position. Mirroring it puts the xSYS2 pin order in line with the bundle, so the header must be fitted from the top when it's needed.
  * The old RST_N stub to the removed v1 header (via at the right-hand board edge) was deleted.
* **Buck converter re-laid in TI's style.**
  * The SW node, VIN and VOUT are small copper polygons.
  * L8 is aligned with the SW pin, so the SW connection is one straight run.
  * Cout sits directly under L8.
  * The feedback trace goes straight down from the FB pin, runs one 45° segment, then chains R74 → C100 → R75 in a single line.
  * The output sense trace goes from Cout to the divider. GND vias sit next to the Cin and Cout returns.
* **Fewer vias**: 41 on the new routes, down from 62.
  * I²S BCLK has 3 vias; LRCK has 3, and joins the existing LRCK line at a via.
  * X1D00 is on one layer with no vias; DIN has 1.
  * **LRCK is still long (~47 mm).** The only short channel out of the XMOS I²S corner has room for one via, and BCLK (the faster clock) uses it.
* **+0V9 feed**: a single 0.5 mm track around the tracks that separate the buck from FB3/C20. There are no vias in it, and it replaces the old loop with two vias.
* **Touch electrode ELEC1** is kept away from the buck and the TAS5805M thermal pad.
* Board silkscreen and title blocks now say **v2.1**.

## Layout bugs found and fixed (present in the v2 files)

* **XU316_RST was never connected** from the ESP to the XMOS reset transistor. Its track also **shorted to ESP GPIO46 (pad 52)**. Rerouted from GPIO21 (see #8).
* **MPR121 IRQ → ESP GPIO2 was unrouted.** Routed.
* **Centre touch electrode (ELEC1) was unrouted** to its ring. Routed.
* The buck converter was placed in the open area right of the mounting hole. The power loop (Cin–PVIN–PGND, SW–L) is routed by hand.
* The +3V3 inner-plane split caused by new vias is bridged on In2. Isolated GND pour islands got stitching vias.

All new routes were checked with KiCad 9 DRC, with schematic parity. The I²S and JTAG escapes at the 0.4 mm-pitch XMOS use 0.127 mm track and clearance, the board minimum. **Those escapes and the LRCK detour are the areas a human reviewer should look at first.**

## Firmware (MiciMike.yaml)

* voice_kit `reset_pin: GPIO21` (v2.1; the original YAML had GPIO4). LED strip `pin: GPIO4` (v2.1; was GPIO21). Hardware mute on GPIO1 (was GPIO33, a PSRAM pin).
* Touch uses the **MPR121** at 0x5A: ELEC0 = left/vol-, ELEC1 = centre/action, ELEC2 = right/vol+. It replaces ESP32 native touch.
* Speaker path matches the Voice PE: 48 kHz, 32-bit, stereo, ESP in secondary mode, data out on GPIO10.
* Added the **TAS5805M** through `github://mrtoy-me/esphome-tas58xx`: address 0x2D, enable GPIO18, **PBTL mono**, `analog_gain: -15.5dB`. PVDD is 14 V and the Nest Mini driver is small, so raise the gain slowly.
* Fixed LED effects that newer ESPHome rejects, and effect loops that wrote 12 LEDs into a 4-LED segment.
* Validated with `esphome config` 2026.6.5. It was not compiled.

## Known / accepted DRC items (all pre-existing in v2)

* MK3 (not fitted): VDD_MIC and MIC_CLK_M are still unrouted to it.
* Touch rings and mic GND pads are at the board edge, by design.
* Antenna pads inside the antenna keep-out.
* Some via pairs are 0.2 mm hole-to-hole.
* One dangling via on the LED1-DOUT net.
* The USB-C shield pins and TP14–16 (touch labels) have no PCB footprints.
* Library/padstack warnings come from the project-local "Onju" footprint library.

## Before ordering

1. Have a PCB designer review the XMOS escape routing, the buck layout and the JTAG runs.
2. In the JLCPCB preview, check part rotations (CPL) and parts with no LCSC number: C74–C77, C80/C90, C82/C91, C86/C87, L6/L7 (Coilcraft XAL4040), R2–R4, R6, R66, R67, U1.
3. Plan how to program the XMOS: either fit J4 (top side) and use an XTAG4, or order the XMOS flash (U11) pre-programmed.
