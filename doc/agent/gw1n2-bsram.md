# GW1N-2: block RAM against the vendor tool

Device `GW1N-2C`, part `GW1N-UV2QN48C7/I6`, IDE **1.9.12.04** (see
[gw1n2-pllo.md](gw1n2-pllo.md) for why that release).

## Method

One primitive per design, with every `INIT_RAM_xx` filled from a hash of the bit
index, built twice: by the vendor tool with `INS_LOC "m" BSRAM_R10[n]`, and by
Yosys + nextpnr + `gowin_pack` with `BEL="XcY9/BSRAM"`. Compared: the init data
region of the bitstream, and the fuses of the `BSRAM_*` tables in the three tiles
of the site (routing fuses differ between the two routers and are left out).

## Sites

Four sites, row 10 of the vendor grid: `BSRAM_R10[0..3]` = columns 1, 4, 13, 16
of row 9 here (tile types 39, 40, 41). Init data and configuration fuses of a
`pROM` match at all four.

## Sweep (site 0)

81 configurations build in both flows; all 81 match in init data and in the
`BSRAM_*` fuses:

- `pROM` 1, 2, 4, 8, 16, 32 bit and `pROMX9` 9, 18, 36 bit, `READ_MODE` 0 and 1;
- `SDPB` 1 to 32 bit and `SDPX9B`, equal and mixed port widths, `READ_MODE` 0
  and 1, `RESET_MODE` `ASYNC`, a non-zero `BLK_SEL`, constant and live low
  address bits;
- `SP` 1, 8, 16, 32 bit and `SPX9`, `WRITE_MODE` 0, 1, 2;
- `DPB` 1, 4, 8, 16 bit and `DPX9B`, `WRITE_MODE` 0 and 1, mixed widths.

Not in that count:

- `DPB` with `WRITE_MODE` 2: the vendor tool refuses it (`PA2122`).
- `SPX9` 36 bit: the vendor tool builds it; nextpnr stops on a hold check of
  `ADB12` with the 25 MHz input on general routing. Not looked into.

## Three differences found, now handled in `gowin_pack.py` for this device

1. **Semi dual port, write port 16 or 18 bit with constant byte enables.** The
   vendor also sets `SDPB_BEHB`/`SDPB_BELB` to `ENABLE` when the read port is 16
   bit or wider. The code left port B disabled.
2. **True dual port.** The vendor sets the four byte enable attributes only when
   both ports are 16 or 18 bit wide and both have constant low address bits. The
   code decided port by port.
3. **Single port 32 or 36 bit, `WRITE_MODE` 2.** The second half's
   read-before-write bit is `(25, 18)` in tile type 41, and the `BSRAM_SP` table
   lists it under attribute 44 value 10 (`SDPB_MODE` = `RBW`), not under
   `SPB_MODE`.

The vendor's rule for the semi dual port byte enables, as observed: they follow
the write port only. Write port 16/18 bit and constant: port A enabled, port B
enabled if the read port is at least 16 bit. Write port 32/36 bit and constant:
both enabled, whatever the read width. Write port narrower, or live low address
bits: none. Whether the read port's low address bits are constant makes no
difference.

All three are limited to `GW1N-2` in the code. Other devices were not checked
against this IDE release.

## Workarounds other devices need

The vendor's synthesis and post-route netlists for `SP`, `SDPB` and `DPB` with
live `CE`, `OCE`, `RESET`, `WRE` and `BLKSEL` connect the primitive's pins
straight to the user's nets: no added logic on `WRE`/`CE` (`NEED_SP_FIX`), on
`BLKSEL` (`NEED_BLKSEL_FIX`) or after the output (`NEED_BSRAM_OUTREG_FIX`). The
chip database sets only `HAS_SP32` for this device.
