# GW1N-2: Q6, Q7 and the constant on X03, X04, X07, X08

Device `GW1N-2C`, part `GW1N-UV2QN48C7/I6`, IDE **1.9.12.04** (see
[gw1n2-pllo.md](gw1n2-pllo.md) for why that release). Tile types 17, 18, 19.

## What the tables say (read from data)

- The wire table (`wire`, table 2) names `Q6` (46) and `Q7` (47) as sources, as it
  does for every GW1N device. With a negative source: `X03`, `X07`, `N260`, `S260`,
  `E260`, `W260` (`Q6`) and `X04`, `X08`, `N270`, `S270`, `E270`, `W270` (`Q7`).
  With a positive source: `A7`, `N830`, `S830`, `E830`, `W830` (`Q6`) and `C6`,
  `N820`, `S820`, `E820`, `W820` (`Q7`).
- A row with a negative source carries one fuse, and every positive row of the
  same wire that does not go through the same first stage has it too. Example,
  `X04`: `-Q7 (4,32)`, `F5 (4,32) (5,34)`, `N272 (4,35) (6,32)`. So the wire is
  on the negative source when that fuse is clear and no other fuse is set, and
  the fuse takes it off.
- `CLS3` (`shortval` 28) has one row on GW1NZ-1 and GW1N-9C, the SSRAM mode. On
  GW1N-2 it has 19: the register attributes of the other slices, on the fuses of
  `CLS2` plus `(22, 31)`, `(23, 3)` for `REG0_REGSET`, `REG1_REGSET`.
- Of the pre-GW5 `.fse` files of this release that Apicula can parse (41), those
  with the register rows are GW1N-1P5, -1P5B, -1P5C, GW1N-2, -2B, -2C, GW1NR-2,
  -2B, -2C, GW1NZ-2B and -2C, in tile types 17, 18, 19. GW1N-4, GW1N-9, GW1NS-4,
  GW1NZ-1, GW2A-18, GW2A-55 and their variants have the single row.

## What the vendor tool does (builds)

- It counts 2016 logic registers, 7 per cell. It accepts
  `INS_LOC "inst" R5C5[3][A]` for a `DFF` and rejects `[3][B]` (CT1094). So
  slice 3 has one register, DFF6, and `Q6` is its output.
- It uses it without being asked: in the two test designs a toggling bit sits in
  DFF6, read through `X03` with no fuse set, and `A6` with LUT6 = `5555`.
- Apart from that, no wire is ever left on `Q6`/`Q7`. In every logic cell, used or
  not, each of `X03`, `X04`, `X07`, `X08`, `N270`, `S270`, `E270`, `W270` that has
  no source has exactly two fuses set: the one of its negative `Q6`/`Q7` row and
  its alonenode fuse. For `X04` that is `(4, 32)`, `(4, 35)`.
- A wire in that state reads as 1: the IDE feeds the `C` inputs of its carry
  chains from `X04` and `X08` in that state (LUT `f000` and `30cc`, which need
  `C` = 1), 9 and 19 inputs in the two builds that were run on hardware, 72 in
  one placed like nextpnr's.
- The clock enable of its plain `DFF`s comes from `X07` or `X08` in that state.
  The only clock enable fuse of a slice is the one for `CEMUX_1=0` or
  `CEMUX_CE=INV`, and it is not set, so the wire is used as it is and has to be 1.
- `E820`, `W820`, `E830`, `W830` take VCC (277) in every cell.

## What goes wrong when Q6/Q7 are taken for VCC

`create_vcc_pips` renames `Q6`/`Q7` to `VCC`, which gives `X03`, `X04`, `X07`,
`X08` and the `x270` wires a VCC pip without fuses, and replaces the fuses of the
real VCC pips of the `x820`/`x830` wires with those of the `Q6`/`Q7` rows.
nextpnr then feeds the `C` input of an ALU from `X04` with no fuse set.

On hardware (two bitstreams, one that worked for comparison):

| Cell | Wire | LUT7 | Result |
|---|---|---|---|
| counter bit, `C` from `X04` | no fuse | unused | counts |
| counter bit 25 of 29, `C` from `X04` | no fuse | the output function | bits 26..28 never change |
| counter bit 23, `C` from `X04` | no fuse | another function | the output stays low |

A simulation of the unpacked bitstreams gives these three results when `X04`
with no fuse follows LUT7 (registered on the clock of slice 2, or not: both
fit), and passes all three when it is 1 or when only `Q6` is a register. So
`Q7` is not a constant either, although the IDE has no register there. What it
is exactly is not known.

## What Apicula does

`create_slice3_vcc_pips` runs for tile types whose `CLS3` table has the register
rows. It drops the pips from `Q6`/`Q7`, which leaves the real VCC pips of the
`x820`/`x830` wires, and gives `X03`, `X04`, `X07`, `X08` a VCC pip with the two
fuses the vendor sets. `A7`, `C6` and the `x270` wires lose their VCC pip.

## Not known, not done

- `X04` and `X08` as a constant are shown by the vendor's carry chains, `X07` by
  its clock enables. `X03` has the mux of `X07` and the vendor sets the same two
  fuses, but nothing in a vendor build depends on its value.

- DFF6 is not a bel. A wire left without fuses is still on `Q6`/`Q7`; nothing
  reads it.
- Whether `N270`, `S270`, `E270`, `W270` in the vendor's two-fuse state read as 1.
  Nothing in a vendor build depends on it.
- What the two fuses the IDE sets in every cell for `A7` (`(12, 11)`, `(12, 14)`)
  and `C6` (`(8, 54)`, `(8, 55)`) select.
- The IDE gives a static source to every unused wire of every cell; `gowin_pack`
  does not on this device.
- Other devices have the same rows, and no register in slice 3: the rename is
  left alone there.
