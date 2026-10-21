# SiliconSmith report: TinyMaix int8 fully-connected layer

**4.04x faster** (860,730 → 213,022 cycles in the measured region) for **+4.8% LUT-equivalent area** on PicoRV32 (rv32im, fast multiplier).

Speedup is in clock cycles on the same core; clock-frequency impact is reported separately when an FPGA place-and-route result is available.

**Wall-clock: 3.76x** = cycle speedup x Fmax ratio on a Tang Nano 9K (GW1NR-9) place-and-route: 71.0 MHz baseline → 66.2 MHz, median of 3 nextpnr seeds (seed noise is about ±8%; per-seed Fmax 68.6, 63.1, 66.2 MHz).

## New instructions

| Mnemonic | What it does | Encoding | Why |
|---|---|---|---|
| `dot4.sum` | 4-element int8 dot product with 32-bit accumulate | custom-0, funct7=0 (`.insn r 0x0B, 0, 0, rd, rs1, rs2`) | Replaces 8 loads (40 cycles), 4 muls (20-24 cycles), 3 adds (9 cycles) = ~69-73 cycles with 2 loads (10 cycles) + 1 custom (5-7 cycles) = ~15-17 cycles per 4 elements. 4.3x speedup on the inner loop. Uses 4x 8x8 multipliers + adder tree (~4 DSPs + LUTs). |

## Verification

### `dot4.sum`: new instruction (node n1)

- ✅ **L1 Builds (Verilator -Wall lint)**: Verilator -Wall: 5 warnings, 0 errors
- ✅ **L2 Matches the C reference**: 1,000,485 vectors · 0 mismatches vs C reference · patched software == original on 5,000 random calls
- ✅ **L3 Formally proven equivalent to an independent spec**: formally equivalent to spec for all 2^64 inputs (11s)
- ✅ **L4 Bit-identical program output**: bit-identical program output
- ✅ **Benchmark**: 320,542 cycles · 2.69x
- ✅ **L5 Area (Gowin synthesis)**: 2,691 LUT+ALU (+4.8%) · 5 DSP

### `dot4.sum`: software polish (node n1.2)

- Safe unrolled dot4
- ✅ **L1 Builds (Verilator -Wall lint)**: hardware unchanged · patched source only calls intrinsics
- ✅ **L2 Matches the C reference**: patched software == original on 5,000 random calls
- ✅ **L4 Bit-identical program output**: bit-identical program output
- ✅ **Benchmark**: 228,382 cycles · 3.77x
- ✅ **L5 Area (Gowin synthesis)**: unchanged hardware · 2,691 LUT+ALU (+4.8%)
- ↪ **prove**: hardware unchanged: formal proofs carry over

### `dot4.sum`: software polish (node n1.2.1)

- Unroll 8x dot4 sum
- ✅ **L1 Builds (Verilator -Wall lint)**: hardware unchanged · patched source only calls intrinsics
- ✅ **L2 Matches the C reference**: patched software == original on 5,000 random calls
- ✅ **L4 Bit-identical program output**: bit-identical program output
- ✅ **Benchmark**: 213,022 cycles · 4.04x
- ✅ **L5 Area (Gowin synthesis)**: unchanged hardware · 2,691 LUT+ALU (+4.8%)
- ↪ **prove**: hardware unchanged: formal proofs carry over

## How to adopt it

1. Add `rtl/*.v` to your design and connect `ss_pcpi` to the core's PCPI port (PicoRV32: `ENABLE_PCPI=1`). Each `ss_insn_*` unit is a pure combinational module (`rs1`, `rs2` → `rd`), so it ports to other cores' custom-instruction interfaces too.
2. Put `include/ss_intrinsics.h` on your firmware include path. Off-target builds fall back to the C reference, so host tests keep working.
3. Replace the original sources with `src/`. The output was verified bit-identical to the original program.

## Nemotron on Nebius Token Factory

| Model | Calls | Tokens in | Tokens out | Cost |
|---|---|---|---|---|
| nemotron-nano | 25 | 6,005 | 1,505 | $0.001 |
| nemotron-super | 72 | 124,558 | 295,803 | $0.304 |
| nemotron-ultra | 5 | 17,086 | 39,393 | $0.135 |
| **Total** | | | | **$0.440** |

## Candidates the search rejected

- `dot4h.sum` (n4): implementation failed: could not produce a valid spec after 3 attempts: RTL does not compile
%Error: /var/tmp/ss-runs/r_w1_contree3/n4/spec2/lint/ss_insn_dot4h_sum.v:20:24: Number is missing value digits: 32'd
   20 |         else if (sum < 32'd-32768)
      |                        ^~~~
        ...
- `bp4.sum` (n6): no speedup (0.86x vs parent 1.00x)
- `mula2.addh` (n5): unit failed after 2 repairs: patched software differs from the original: FAIL size=9 mode=1 expected=49026 got=-16510 first_bytes_a=-128,127,127,127,-128,127,-128,-128, first_bytes_b=-128,1
- `mula4.add` (n3): unit failed after 2 repairs: patched software differs from the original: FAIL size=9 mode=1 expected=49026 got=-16510 first_bytes_a=-128,127,127,127,-128,127,-128,-128, first_bytes_b=-128,1
- `dot2.sum` (n2.1): no area saving (2,598 vs 2,598 LUT+ALU)
- `dot4.sum` (n1.1): no area saving (2,691 vs 2,691 LUT+ALU)
- `dot2.sum` (n2.2.1): no speedup (1.87x vs parent 1.96x)
- `dot4.sum` (n1.2.1.1): no speedup (3.74x vs parent 4.04x)
- `dot8.sum` (n1.2.1.2): implementation failed: could not produce a valid spec after 3 attempts: RTL disagrees with C reference: rs1=0x00000080 rs2=0x00000080 expected=0x00008000 got=0x00004000
SS_MISMATCH rs1=00000080 rs2=00000080 expected=00008000 got=00004000

- `requant.s` (n1.2.1.4): implementation failed: could not produce a valid spec after 3 attempts: RTL disagrees with C reference: rs1=0x00000000 rs2=0x00000000 expected=0x00000000 got=0xffffff80
SS_MISMATCH rs1=00000000 rs2=00000000 expected=00000000 got=ffffff80

- `zip2.hi` (n2.2.5): unit failed after 2 repairs: patched software does not compile on the host
- `dot4.sum.add` (n1.2.1.3): unit failed after 2 repairs: patched software differs from the original: FAIL size=8 mode=0 expected=18944 got=15314 first_bytes_a=73,-21,29,-15,-33,-118,27,45, first_bytes_b=60,109,81,54,-
- `mula4.add` (n1.2.1.5): unit failed after 2 repairs: patched software differs from the original: FAIL size=4 mode=0 expected=-2404 got=-125501928 first_bytes_a=43,120,19,-20, first_bytes_b=-56,16,-84,16,
- `requant.mulh.rnd.shr` (n2.2.3): unit failed after 2 repairs: patched software differs from the original: FAIL size=4 mode=0 expected=-2404 got=0 first_bytes_a=43,120,19,-20, first_bytes_b=-56,16,-84,16,
- `dot3.sum` (n2.2.2): unit failed after 2 repairs: patched software differs from the original: FAIL size=3 mode=3 expected=127 got=-128 first_bytes_a=-128,127,-128, first_bytes_b=1,1,-1,
- `zip2.lo` (n2.2.4): unit failed after 2 repairs: patched software differs from the original: FAIL size=8 mode=0 expected=18944 got=3630 first_bytes_a=73,-21,29,-15,-33,-118,27,45, first_bytes_b=60,109,81,54,-1

## Provenance

Workload: `tm_dot_prod` from https://github.com/sipeed/TinyMaix (`src/arch_cpu.h`, commit 0532ece, Apache-2.0).
