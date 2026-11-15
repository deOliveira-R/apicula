# GW1N-2 PLLO

The PLL of the GW1N-2 die (GW1N-LV2/UV2 parts, also sold as GW1N-1P5) is the `PLLO`
primitive, not an `rPLL`. Checked with IDE **1.9.12.04** (the only release at hand
that is licensed for the GW1N-2C parts), device `GW1N-2C`, part
`GW1N-UV2QN48C7/I6`. Nothing here has been run on hardware.

The vendor handler is `fusegenMem::setPllAttrInPll_1N2` (see [pll.md](pll.md)). It
was not read: everything below comes from the vendor tables and from bitstreams
built with the IDE. "Build" means an IDE build made for the purpose, decoded with
the tables of the chip database.

## Cells

Read from tables:

- The fuses are in tile type 88 at (row 1, col 0): `shortval` table `PLL` (35),
  1351 rows, 178 fuses. Every row has a single key, so the cell decodes exactly:
  the value of an attribute is the one whose fuses are set. Seven rows have a
  negative key (the fuse of the value `ENABLE` of `SRSTEN`, `RSTEN`, `PWDEN`,
  `PLOCK`, `IRSTEN`, `FLTOP`, `FLOCK` is set unless `DISABLE` is given).
- The corner cell (0, 0), tile type 50, has seven more `PLL` rows: `PLLREG0..2`,
  `PLLVCC1_TRIM0/1`, `VCCREG_TRIM0/1`.
- The vendor grid marks the corner (type 50) as the PLL cell (`P`), and
  `dat.portmap['PllClkin']` counts rows from it: `(F6, 4)` is the input of the
  `IOL5A` pad, `(CLK0, 0)` the routed clock input. So the bel is at (0, 0) and
  its fuses are in the cell below.
- `PllIn`, `PllOut` are empty for this device. The ports were found by builds.

## Ports

Found by builds. Inputs sit on consecutive `A0..D7` wires of three cells:

| Cell | Wires | Ports |
|---|---|---|
| (0, 0) | `A3`..`D7` | `IDSEL[0..5]`, `FBDSEL[0..5]`, `ODSELA[0..6]`, `ODSELB[0..6]`, `ODSELC[0..2]` |
| (0, 0) | `CLK0`, `CLK2` | `CLKIN`, `CLKFB` |
| (1, 0) | `A2`..`B1` | `RESET`, `RESET_P`, `RESET_I`, `RESET_S`, `ENCLKA..D` |
| (1, 0) | `B3`..`D6` | `DTA[0..3]`, `DTB[0..3]`, `ICPSEL[0..4]`, `LPFRES[0..2]`, `PSDIR`, `PSSEL[0..1]`, `PSPULSE` |
| (2, 0) | `A0`..`B2` | `ODSELC[3..6]`, `ODSELD[0..6]` |
| (0, 0) | `F2`, `F4`..`F7` | `LOCK`, `CLKOUTA..D` |

Method: the dynamic inputs were tied to 0 or 1 by a 7-bit code and its complement
(14 builds), `IDSEL`/`FBDSEL` again with their dynamic selects on; `ENCLKx` by
driving one of them from a pin; `LOCK` by leaving it open; the clock outputs
with `CLOCK_LOC "net" LOCAL_CLOCK`, which keeps a net off the clock network
(otherwise the IDE sends `CLKOUTA`/`CLKOUTB` to logic through the clock network
and their wires never show).

The outputs enter the clock muxes as `TRPLL0CLK0..3` = `CLKOUTA..D` (one build per
output). The name is positional nonsense for a PLL in the top left corner; the
same goes for `PCLKR1` and `TLBDCLK2` in this table.

Clock input (builds):

| Source | `INSEL` | Path |
|---|---|---|
| `IOL5A` (`LPLL_T_IN`, QN48 pin 16) | `CLKIN0` | no routing fuses |
| any routed net | `CLKIN1` | `CLK0` of (0, 0) |
| `IOL6A` (`GCLKT_7`, QN48 pin 18) | `CLKIN4` | one fuse at (9, 0), bit (25, 26), in no table Apicula reads |

Feedback (builds): `CLKFB_SEL="INTERNAL"` is `FBSEL=CLKFB3`, and `CLKFB` is not
routed. `EXTERNAL` from a routed net is `FBSEL=CLKFB0` on `CLK2` of (0, 0); this
is also what the IDE does for `IOL4A` (`LPLL_T_FB`). `EXTERNAL` with `CLKFB` tied
to a constant is built as internal feedback. From `IOL6A` no `FBSEL`
fuse is set and six fuses of (9, 0), row 24, are. The two `IOL6A` paths are not
implemented: nextpnr routes such a clock to `CLK0`/`CLK2`.

The IDE does not route `IDSEL`/`FBDSEL` unless `DYN_IDIV_SEL`/`DYN_FBDIV_SEL` is
`"TRUE"`. nextpnr disconnects them, and `CLKFB` for internal feedback, to get the
same routing.

## Parameters to attributes

Checked by builds, one or more per line.

| Parameter | Attribute |
|---|---|
| `IDIV_SEL`, `FBDIV_SEL` | `IDIV`, `FDIV` = value + 1; 64 has no fuses |
| `ODIVx_SEL` | `DIVx` = value; 128 has no fuses |
| `DYN_IDIV_SEL`, `DYN_FBDIV_SEL`, `DYN_ODIVx_SEL` | `IDIVSEL`, `FDIVSEL`, `DIVx_SEL` = `DYN` |
| `CLKOUTx_EN` | `CKx=ENABLE`; follows the parameter, not the connection |
| `CLKx_IN_SEL` | `CKx_IN`: `0x` `VCO`, `10` `CASCADE` (`VCO` on A), `11` `BYPASS` |
| `CLKx_OUT_SEL=1` | `CKx_OUT=BYPASS` |
| `PSx_COARSE` (B, C, D) | `PSx_COARSE` = value; 128 has no fuses |
| `PSx_FINE` (B, C, D) | `PSx_FINE` = value + 1; 7 has no fuses |
| `DYN_PSx_SEL`, `DTMS_ENx` | `PSx_SEL=DYN`, `DTMS_ENx=ENABLE` |
| `DYN_DPA_EN` | `DPAEN=ENABLE` |
| `CLKOUTx_DT_DIR`, `CLKOUTx_DT_STEP` (A, B) | `DUTY_TRIM_x`: `P0`, `P50`, `P100`, `P200` for steps 0, 1, 2, 4 with direction 1, `M0`..`M200` with direction 0 |
| `DYN_DTx_SEL` | `DTx_SEL=DYN` |
| `RESET_I_EN`, `RESET_S_EN` | `IRSTEN`, `SRSTEN` = `ENABLE` or `DISABLE` |
| `ICP_SEL` | `ICPSEL` = (value + 1) * 10; 0 has no fuses |
| `LPR_REF` (one hot) | `LPR` = `R1`..`R7`; `7'b0000000` (R0) has no fuses |
| `DYN_ICP_SEL`, `DYN_RES_SEL` | `ICPDYN_SEL`, `LPR_SEL` = `DYN` |
| `FCLKIN` | `FLDCOUNT`, from `FCLKIN / (IDIV_SEL + 1)` as for the rPLL |

Set for every PLLO: `PSA_COARSE=1`, `PSA_FINE=1`, `FLOCK=ENABLE`, `FLTOP=ENABLE`,
`PLOCK=DISABLE`, `GMCMODE=15`, `PDN=ENABLE`, `PWDEN=ENABLE`, `RSTEN=ENABLE`. In the
corner cell `PLLREG0..2=ENABLE`; the IDE sets those three in designs without a
PLL as well, `gowin_pack` only with one.

`ICP_SEL` and `LPR_REF` left at `X` (the default): the IDE computes them. The
GW1N rPLL formula (`get_pll_pump`) gives the IDE's values when the VCO frequency
is taken as `FCLKIN / (IDIV_SEL + 1) * (FBDIV_SEL + 1) * ODIVA_SEL`: 29 builds
(FCLKIN 3 to 400 MHz, feedback dividers 1 to 64, `ODIVA_SEL` 1 to 128, input
dividers 1, 4, 8). Each of the two is computed on its own when only the other is
given (2 builds). `FLDCOUNT` steps at 30, 60, 90 MHz as the formula says (13
builds).

Rejected by the IDE, and by `gowin_pack` (the IDE's message in brackets):

- PFD frequency outside 3 to 400 MHz (PA2078).
- `PSx_COARSE` greater than `ODIVx_SEL` (EX0316).
- `PSx_FINE` not 0 with `CLKx_IN_SEL` `1x` (EX0330).
- `DTMS_ENx="TRUE"` without `DYN_DPA_EN="TRUE"` (EX0330).
- `CLKA_IN_SEL=2'b11` with another `CLKx_IN_SEL` of `0x` (EX0322).

Rejected by the IDE only: `CLKOUTx_EN="FALSE"` or `ENCLKx` tied low with `CLKOUTx`
connected (PA2145, CK3001). The IDE warns, and no more, about output frequencies
outside 3.125 to 800 MHz (PA1019); `gowin_pack` checks no output or VCO frequency.

## Verification

232 IDE builds were rebuilt with Yosys 0.71, nextpnr-himbaechel and `gowin_pack`,
with every pin constrained to the IDE's choice, and compared with the IDE's
bitstream:

- fuses: all 185 fuse positions of the `PLL` tables of (1, 0) and (0, 0), bit for bit;
- inputs: what drives each of the 70 input wires (VCC, VSS, a net, or nothing),
  by tracing both bitstreams back through the routing;
- outputs: which `TRPLL0CLKn` enter the clock network and which `F` wires of
  (0, 0) are used.

The builds: a base and 139 variants of it (one parameter changed in most, a few
together in some; each parameter of the primitive, several values), 19 more for
combinations and the cases the first set missed, 25 for `FCLKIN` and the computed `ICP_SEL`/`LPR_REF`, 22 with
coded constants on the dynamic inputs, and 26 for clock input, feedback, enables
and outputs.

| Result | Builds |
|---|---|
| All three equal | 222 |
| Fuses and inputs equal; outputs not comparable: a PLL output used as data goes through the clock network or its `F` wire, and the tools choose differently | 8 |
| Differ: clock or feedback from `IOL6A`, not implemented (above) | 2 |

The input check shows that the same wires are driven and with the same constant;
it does not identify the net behind a wire that a signal drives.

The other bits are not compared one by one. In the base build the three cells
differ in routing muxes only (the IDE ties unused wires and takes other routes
for the constants), plus 2 `CFG` and 4 `BANK` fuses of the corner. Over the whole
chip, leaving routing muxes aside, the base build differs from the IDE's in the
same kinds of fuses as a counter without a PLL does (IO defaults such as `CLAMP`
on input pads, `CFG`, `BANK`, logic placement). So the input from `IOL5A` and the
entry into the clock muxes take no fuse outside the three cells.

`gowin_unpack` gives back the parameters of the source for all 232 IDE bitstreams
and for the 232 made by `gowin_pack`, except `CLKFB_SEL` in the build with a
constant feedback input (built as internal by both). `FCLKIN` is not stored in a
bitstream: the unpacker writes the top of the PFD range that `FLDCOUNT` stands
for, times the input divider.

Designs without a PLLO: the same nextpnr JSON packs to byte-identical bitstreams
with the commit before this work and after it, for GW1N-2 (2 designs) and GW1NZ-1
(4 examples, one with an rPLL); `gowin_unpack` of three of them gives the same
netlist. By `tools/chipdb_compare.py` the databases of GW1NZ-1 and GW1NS-4 are
identical before and after, and those of GW1N-4, GW1N-9 and GW1N-9C but for NaN
timing values that both have (this IDE release; nextpnr cannot use them, which is
why GW1NZ-1 stands in for GW1N-9C). The GW1N-2 one differs by the new bel, its 39
alias nodes, the `F` wires joined to the four `TRPLL0CLKn` nodes and the pad
entry. The other databases do not build from this IDE release, before or after.

## Not known

- Whether a bitstream made this way locks on hardware.
- What `CLKMUX_CLKIN1`, `CLKMUX_CLKIN2`, `CLKMUX_CLKFB0`, `SELIN`, `INTFB`, `MON`,
  `KVCO`, `GMCGAIN`, `GMCOUT`, `GMCVREF`, `CRIPPLE`, `RSTLF` and the trims of the
  corner cell do; no build set them.
- The meaning of `INSEL=CLKIN2/CLKIN3` and `FBSEL=CLKFB1/2/4`.
- The fuses of (9, 0) that carry `IOL6A` to the PLL.
- Wire `B2` of (1, 0), between `ENCLKD` and `DTA[0]`: never driven by the IDE.
- Yosys 0.71 has no `PLLO` in its gw1n cell library; a blackbox declaration with
  the vendor's port list is needed. It does not pass default parameter values
  on, so `gowin_pack` carries the defaults of the vendor's simulation model.
