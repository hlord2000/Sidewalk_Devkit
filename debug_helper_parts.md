# USB Debug Helper Parts

Date: 2026-05-28

## Chosen MCU

Use `STM32G0B1KBU6`.

- LCSC: `C5159549`
- Package: `UFQFPN-32 5x5`
- Function: USB-powered Zephyr debug helper MCU
- Relevant fit: Zephyr `SOC_STM32G0B1XX`, USB FS from HSI48, UART, enough GPIO for SWD bit-bang
- Parts MCP availability: 325 in stock, extended part, about `$3.02` qty 1 / `$2.55` qty 10

## Target Boundary Interface

Use a `TXU0204RUTR` fixed-direction translator, a dedicated `TXU0101DRYR`
SWDIO output translator, and a switched copy of the target `VTREF`.

SWCLK is one-way helper-to-target. SWDIO is bidirectional at the protocol level,
but this circuit handles it as two one-way legs tied together on the target
side: a target-to-helper read path and a separately enabled helper-to-target
drive path. That matches Zephyr's `dio-gpios`, `dout-gpios`, and `dnoe-gpios`
model.

One `TXU0204` cannot safely do the whole boundary by itself because its single
`OE` would also enable the helper-to-target SWDIO driver during target reads.
Use `TXU0101DRYR` as the independently enabled SWDIO output buffer.

### Recommended Parts

`U_LS_MAIN = TXU0204RUTR`

- LCSC: `C5217875`
- Package: `UQFN-12 1.7x2`
- Quantity: `1`
- Function: dual-supply 4-bit fixed-direction translator, two channels A to B and two channels B to A
- Relevant fit: SWCLK, UART TX, UART RX, and SWDIO read path
- Parts MCP availability: 1354 in stock, extended part, about `$0.90` qty 1 / `$0.78` qty 10

`U_LS_SWDIO_OUT = TXU0101DRYR`

- LCSC: `C5217869`
- Package: `SON-6 1x1.4`
- Quantity: `1`
- Function: dedicated one-channel translator used as a separately enabled SWDIO output buffer
- Relevant fit: helper-driven SWDIO output; high-Z whenever Zephyr's `dnoe-gpios` is inactive
- Parts MCP availability: 242 in stock, extended part, about `$0.50` qty 1 / `$0.43` qty 10

`U_VTREF_SW = TPS22917DBVR`

- LCSC: `C2681320`
- Package: `SOT-23-6`
- Quantity: `1`
- Function: low-leakage load switch for `TARGET_VTREF_NPM` to `VTREF_DBG`
- Relevant fit: 1 V to 5.5 V input, active-high ON, reverse-current blocking, low off current
- Parts MCP availability: 19645 in stock, extended part, about `$0.31` qty 1 / `$0.23` qty 50

`Q_RESET = 2N7002`

- LCSC: `C8545`
- Package: `SOT-23`
- Quantity: `1`
- Function: debugger-controlled open-drain `nRESET` pull-down
- Parts MCP availability: basic part, about 1.63M in stock, about `$0.014` qty 1

## Power And Leakage Rule

The helper and target stay separate power domains.

- `DBG_3V3`: helper-side rail from the helper USB port; powers the STM32 and all translator `VCCA` pins.
- `TARGET_VTREF_NPM`: target IO rail from the nPM1300; do not connect this directly to translator `VCCB`.
- `VTREF_DBG`: switched copy of `TARGET_VTREF_NPM`; powers translator `VCCB` only while helper USB is present and the helper firmware enables the boundary.

Wire `TPS22917DBVR` as:

- `VIN`: `TARGET_VTREF_NPM`
- `VOUT`: `VTREF_DBG`
- `ON`: `DBG_BOUNDARY_EN` from the helper, with about `100 kOhm` pulldown to ground
- `CT`: open, unless a slower `VTREF_DBG` rise is wanted
- `QOD`: connect to `VOUT` for fast `VTREF_DBG` discharge, or leave open for the slowest discharge

When helper USB is unplugged, `DBG_BOUNDARY_EN` is low, `VTREF_DBG` is off, and the target rail is not feeding translator supply current. The remaining target-side current is only pin leakage into unpowered translator I/O plus the load switch off leakage.

Worst-case leakage estimate with helper USB absent:

- `TXU0204` / `TXU0101` Ioff max: `2.5 uA` per A/B pin over `-40 C` to `125 C`
- Target-connected translator pins: `SWCLK`, target `UART_RX`, target `UART_TX`, `SWDIO_IN`, `SWDIO_OUT` = `5` pins
- Translator pin leakage budget: about `12.5 uA` worst case across temperature
- `TPS22917` shutdown current: `250 nA` max to `105 C`; reverse leakage is `1 uA` max
- Add the selected reset MOSFET's off leakage from its own datasheet if reset leakage is critical

Typical leakage should be much lower, but the guaranteed worst-case number is what should be used for battery-life budgeting.

Add local decoupling: `100 nF` from each translator `VCCA` to ground, `100 nF` from each translator `VCCB` to ground, and `100 nF` to `1 uF` on `VTREF_DBG` after the load switch.

## Signal Mapping

Define side A as the STM32 debug helper and side B as the nRF54L15 target.
Helper and target grounds must be common at the debug boundary.

`U_LS_MAIN = TXU0204RUTR`

- `VCCA`: `DBG_3V3`
- `VCCB`: `VTREF_DBG`
- `OE`: `DBG_BOUNDARY_EN`, active high; add about `100 kOhm` pulldown to ground
- `A1 -> B1Y`: `DBG_SWCLK` to `TARGET_SWDCLK`
- `A2 -> B2Y`: `DBG_UART_TX` to `TARGET_UART_RX`
- `B3 -> A3Y`: `TARGET_UART_TX` to `DBG_UART_RX`
- `B4 -> A4Y`: `TARGET_SWDIO` to `DBG_SWDIO_IN`

If `U_LS_MAIN.OE` stays high while Zephyr has the SWD port off, add a weak
pulldown around `100 kOhm` on `DBG_SWCLK` so the target clock does not float.
Firmware should also leave `DBG_SWCLK` low when idle.

`U_LS_SWDIO_OUT = TXU0101DRYR`

- `VCCA`: `DBG_3V3`
- `VCCB`: `VTREF_DBG`
- `A -> BY`: `DBG_SWDIO_OUT` to `TARGET_SWDIO`
- `OE`: `DBG_SWDIO_OE`, active high; add about `100 kOhm` pulldown to ground

The target `SWDIO` node intentionally connects to both `U_LS_MAIN.B4` and
`U_LS_SWDIO_OUT.BY`. `U_LS_MAIN.B4` is only the target-to-helper read path.
`U_LS_SWDIO_OUT.BY` is high-Z except when Zephyr is actively driving SWDIO.

`Q_RESET = 2N7002`

- Drain: `TARGET_nRESET`
- Source: ground
- Gate: `DBG_nRESET_DRIVE_LOW` from the STM32 through about `100 Ohm`
- Gate pulldown: about `100 kOhm` to ground
- Target reset pullup: to the target IO rail, normally about `10 kOhm`

This keeps reset open-drain. When helper USB is unplugged, the MOSFET gate is held low and the target reset pullup does not back-power the helper.

Consider `22` to `47 Ohm` series resistors near the translator outputs on `SWCLK` and `SWDIO_OUT` if routing is longer than a short local connection.

## Zephyr SWDP Notes

Use Zephyr's GPIO SWDP binding with separate SWDIO input and output:

- `clk-gpios`: `DBG_SWCLK`
- `dio-gpios`: `DBG_SWDIO_IN`
- `dout-gpios`: `DBG_SWDIO_OUT`
- `dnoe-gpios`: `DBG_SWDIO_OE` configured as `GPIO_ACTIVE_HIGH`
- `noe-gpios`: omit for the `TXU0204` shared main translator, or use only if you accept that it also gates UART
- `reset-gpios`: `DBG_nRESET_DRIVE_LOW`; choose the GPIO polarity so Zephyr's deasserted reset state drives the NMOS gate physically low
- `keep-reset-deasserted`: include this property if your Zephyr tree has it and you want the driver to actively hold reset released after `swdp_port_off()`

The `dnoe-gpios` name is historical in Zephyr; the driver drives it high while outputting SWDIO and low while reading. That maps directly to the active-high `OE` pin on `TXU0101DRYR`.

If `noe-gpios` is used, the driver drives it logical active while the SWD port is on and logical inactive while off. Do not connect it to `U_LS_MAIN.OE` unless it is acceptable for UART to disappear whenever Zephyr turns the SWD port off.

Reset polarity needs one board-level check because Zephyr trees differ here. With the NMOS reset pull-down above, the safe electrical state is gate low. The `100 kOhm` gate pulldown guarantees that state when helper USB is absent. In firmware, configure the GPIO polarity so Zephyr's deasserted reset state produces a physical low on `DBG_nRESET_DRIVE_LOW`.

## Rejected Shortcut

Do not use a passive or auto-direction level shifter for SWDIO. SWDIO is push-pull and turnaround-sensitive, and Zephyr already supports the explicit `dio-gpios` / `dout-gpios` / `dnoe-gpios` pattern. The `TXU0204` plus `TXU0101` split follows that pattern directly.

`SN74AXC4T774RSVR` is tempting because it is a tiny per-bit-direction translator, but it does not give a separate OE for only the SWDIO output channel. It is less clean with the stock Zephyr SWDP binding than the `TXU0101` output-enable path.

Do not try to use only one `TXU0204` for both SWDIO directions. Its single `OE` would enable the SWDIO output driver during reads and can fight the target.

A second `TXU0204` for `SWDIO_OUT` also works electrically, but it wastes three channels and more board area compared with `TXU0101DRYR`.

## Sources Checked

- Parts MCP: `STM32G0B1KBU6`, `TXU0204RUTR`, `TXU0101DRYR`, `TPS22917DBVR`, `2N7002`, `SN74AXC4T774RSVR`
- Local Zephyr tree: `stm32g0b1` USB/HSI48 support, `zephyr,swdp-gpio` binding, and `swdp_bitbang.c`
- TI TXU0204 / TXU0101 product pages and datasheets: active-high `OE`, 1.08 V to 5.5 V rails, VCC isolation, and Ioff partial-power-down
- TI TPS22917 datasheet/product page: active-high ON, ultra-low shutdown current, and reverse-current blocking
