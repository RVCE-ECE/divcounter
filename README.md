![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg) ![](../../workflows/test/badge.svg) ![](../../workflows/fpga/badge.svg)

# Glitch-Free Divider + Gated Frequency Counter

A Tiny Tapeout ASIC (SKY130, 1x1 tile, Verilog) combining a glitch-free programmable clock divider (N = 2..255) with a gated frequency counter and SPI serial readout.

- Top module: `tt_um_janane2802_divcounter`
- Clock: 50 MHz
- Full documentation: [docs/info.md](docs/info.md)

## Team

Sri Janane S V, Varsha M, Nishita Singhvi, Shreya and Shylashree N, RV College of Engineering (RVCE), Bengaluru.

We acknowledge CoE-ICAS and the Department of ECE, RVCE, for the resources. Special thanks to Dr. H V Ravish Aradhya (HoD, ECE), Dr. K R Usha Rani (Associate Dean - PG), Dr. K S Geetha (Vice Principal) and Dr. K N Subramanya (Principal).

## What it does

- **Divider mode:** divides the input clock by a programmable N (2..255) and drives `clk_out`. Ratio changes are glitch-free: a new N is held in a shadow register and only takes effect at the next period boundary, so no runt pulses appear on the output.
- **Frequency counter mode:** a prescaler counts edges of an external signal on `sig_in`. A gate timer opens a measurement window of 1024 to about 16.7 million clock cycles. The result is read out as a 24-bit frame over SPI (mode 0).
- **Single clock domain:** no derived clocks and no ripple dividers. Every external control pin goes through a 2-flip-flop synchronizer.

Frequency from a frame: `f_in = count * N_used * f_clk / window`, where `window = 2^(10 + 2*gate_sel)` clock cycles.

## Pinout

| Pin | Direction | Signal |
|---|---|---|
| `ui_in[7:0]` | in | `data_bus[7:0]` (register write data) |
| `uo_out[0]` | out | `clk_out` |
| `uo_out[1]` | out | `miso` (SPI data out) |
| `uo_out[2]` | out | `busy` |
| `uo_out[3]` | out | `done` |
| `uo_out[4]` | out | `overflow` |
| `uo_out[5]` | out | `ratio_pending` |
| `uo_out[6]` | out | `ratio_applied` |
| `uo_out[7]` | out | `gate` |
| `uio[0]` | in | `wr` (write strobe) |
| `uio[1]` | in | `reg_sel` (0 = N register, 1 = control register) |
| `uio[2]` | in | `sig_in` (signal to measure) |
| `uio[3]` | in | `sck` (SPI clock) |
| `uio[4]` | in | `cs_n` (SPI chip select, active low) |
| `uio[7:5]` | - | unused |

## How to test

1. Hold `rst_n` low for a few clock cycles, then release it. Keep `cs_n` high when not reading.
2. **Divider:** set `reg_sel=0`, put N on the data bus, pulse `wr` high for at least 3 clock cycles, then drop it. `clk_out` divides by N once the write commits.
3. **Frequency counter:** write the control register (`reg_sel=1`) with `mode=1` and the desired `gate_sel`, apply the signal to `sig_in`, then write the control register again with `arm=1`. Wait for `done`, then read the 24-bit frame `{N_used, count}` over SPI and compute `f_in`.
4. Check `overflow`: if set, `count` is saturated at `0xFFFF`.

The register map, write protocol, worked example, and limitations are in [docs/info.md](docs/info.md).

## Verification

- cocotb test suite for RTL and gate-level simulation (`test/`), plus an iverilog-based exhaustive runt-pulse checker for glitch-free switching.
- Hardened with LibreLane (SKY130A): DRC, LVS and antenna checks clean; setup/hold clean at all signoff corners.
- Known limitation: max-slew violations at the slow, hot, low-voltage corner only (details in docs/info.md).
- Tiny Tapeout precheck: 14 of 15 checks pass. The remaining Magic DRC failure is a local tool-version issue; DRC is clean inside LibreLane.

## Resources

- [Tiny Tapeout](https://tinytapeout.com)
- [FAQ](https://tinytapeout.com/faq/)
- [Build your design locally](https://www.tinytapeout.com/guides/local-hardening/)
