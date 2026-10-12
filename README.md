# Croco Krave — Byte-Level Hardware SECDED for the Croc SoC

**A RISC-V SoC hardened against radiation-induced bit flips, taken from RTL to GDSII in IHP 130 nm.**

An in-memory SECDED (Single Error Correction, Double Error Detection) codec replaces the equivalent
software routine, making every load and store self-correcting at **zero cycle cost** — error
correction happens inside the existing memory-access cycle instead of costing hundreds of cycles
of CPU bit-manipulation.

ETH Zürich — VLSI 2 (FS 2026) · Alessandro Vegni & Giulio Guerrieri · Group 14
Full technical write-up: **[Final_Report.pdf](doc/Final_Report.pdf)**

![SECDED Block Diagram](doc/Final_Block_Diagram.png)

---

## Headline results

| | Result |
|---|---|
| **Memory-access speedup vs. software SECDED** | **up to 44.0× fewer cycles · 32.1× faster in wall-clock time** |
| **Correction latency** | 0 extra cycles — fully combinational, inside the memory cycle |
| **Codec verification** | 23,552 exhaustive test vectors, **100% pass, zero mismatches** |
| **Sign-off** | clean DRC (KLayout), clean LVS (Calibre), **zero hold violations** |
| **Power cost** | +8.15% total (18.40 → 19.90 mW) |
| **Area cost of the fault-tolerance logic** | repair buffers = **1% of total die area** |

### Hardware vs. software SECDED

Both configurations were measured on the real design: the software baseline runs on the unmodified
Croc (`SECDEDBypass = 1`, native 32-bit SRAM, no pipeline cut), the hardware path on the full
accelerated architecture. Cycle counts are measured end-to-end via MMIO start/stop markers, so
they include bus transaction overhead (7 cycles baseline, 15 cycles with the `obi_cut` stage).

| Scenario | SW cycles | HW cycles | Cycle speedup | SW time @100 MHz | HW time @73 MHz | Time speedup |
|---|---:|---:|---:|---:|---:|---:|
| Encode / Store | 267 | 15 | **17.8×** | 2.67 µs | 0.21 µs | **13.0×** |
| Decode / Load (0 errors) | 518 | 15 | **34.5×** | 5.18 µs | 0.21 µs | **25.2×** |
| Decode / Load (1 error) | 657 | 15 | **43.8×** | 6.57 µs | 0.21 µs | **32.0×** |
| Decode / Load (2 errors) | 660 | 15 | **44.0×** | 6.60 µs | 0.21 µs | **32.1×** |

The hardware path costs a **constant ~205 ns per transaction regardless of the error state**,
while the software routine degrades as soon as it has to locate and repair a flipped bit
(syndrome recomputation, weight calculation, branches, correction mask).

The speedups above are reported *after* paying the honest cost of the design: the longer SECDED
datapath forced the clock down from 100 MHz to 73 MHz, and the figures are computed at those two
different frequencies. The hardware still wins by more than an order of magnitude.

Software SECDED also has a limit no amount of optimization fixes: **it cannot protect the memory
holding the bootloader, the core program, or the correction routine itself.** A bit flip there
compromises the handler before it can run. The hardware codec covers the entire memory map from
the first boot cycle.

---

## What was built

### Byte-level Hsiao codec in the memory path
Each 8-bit byte is independently encoded into 13 bits (8 data + 5 parity) using an **Inverted
Hsiao (13, 8)** code. Hsiao was chosen over classic Hamming for its balanced XOR tree — shorter
critical path and lower gate count.

Encoding at **byte granularity rather than word granularity** is the key architectural decision:
it preserves native RISC-V byte addressability, so a single-byte store executes directly instead
of requiring a Read-Modify-Write cycle with its own FSM and latency penalty. The cost is four
parallel encoder/decoder pairs per bank (eight codec units total) — a good trade on a pad-limited
die with surplus area.

### Transparent drop-in wrapper
`secded_sram_impl` replaces `tc_sram_impl` behind an unchanged 32-bit interface. The SRAM was
widened to 1024×64; the core, crossbar and peripherals are untouched and see identical data.
Double-bit errors are OR-aggregated and raised on the core's `interrupts_o`, trapping the fault
before corrupted data is consumed.

### Silent repair buffer (scrubbing without stalls)
Correcting a bit flip on the read path does **not** fix the bit still rotting in the SRAM — a
second flip in the same byte would turn a correctable error into an uncorrectable one. The
`secded_repair_buffer` latches the corrected word and writes it back during the next idle cycle,
with collision detection that invalidates the pending writeback if the address is overwritten
first. The CPU never stalls, and the writeback touches only the corrupted byte(s).

### Timing closure via inter-domain pipeline cut
The combinational codec pushed the core↔memory path into negative setup slack. An `obi_cut` spill
register stage was inserted at the `croc_domain` / `user_domain` boundary, splitting the path into
two pipeline segments and closing timing at the cost of one cycle of bus latency.

### Silicon-level testability
Two GPIO pads expose the per-bank double-error flags; three input pads drive an on-chip fault
injection engine (per-bank target + single/double fault select), synchronized across the domain
boundary. Faults can be injected into the fabricated chip from the outside.

### Reversible by one parameter
`SECDEDBypass` in [`rtl/user_pkg.sv`](rtl/user_pkg.sv) structurally switches the whole design
between the original Croc baseline (32-bit SRAM, no codec, no pipeline cut) and the hardened
implementation — which is exactly what made the honest A/B benchmark above possible.

---

## Verification

**Exhaustive unit-level proof.** The 8-bit input space is small enough to verify completely rather
than sample randomly. `tb_secded_byte` sweeps every data word against every error vector:

```
N = 2⁸ × (1 clean + C(13,1) single + C(13,2) double) = 256 × (1 + 13 + 78) = 23,552 vectors
```

All 23,552 pass with zero mismatches: clean roundtrip raises no false flags, all 3,328 single-bit
injections are located and bit-accurately repaired, all 19,968 double-bit injections assert
`double_err_o`.

**System level.** `bitflip_verify.c` runs on the core inside `tb_fault_injection_croc.sv` with
faults XOR-injected into the SRAM; both the single-error flag and the double-error interrupt
assert correctly end-to-end.

**Repair buffer.** `sec_repair.c` in `tb_sec_repair.sv` writes a corrupted line, reads it (flag
asserts, data corrected, writeback scheduled), idles, then re-reads — the flag stays low,
confirming the SRAM itself was silently repaired.

---

## Physical implementation (IHP 130 nm, open-source flow)

Yosys → OpenROAD → KLayout → Calibre, signed off at the tt corner, 1.20 V, 25 °C.

| Metric | Original Croc | Croco Krave |
|---|---:|---:|
| Die area | 314,200 µm² | 861,500 µm² |
| — memory macros | 125,400 µm² | 529,213 µm² |
| — standard cells | 188,800 µm² | 332,287 µm² |
| Total power | 18.40 mW | 19.90 mW (+8.15%) |
| f_max post-route | 100 MHz | 73.5 MHz |
| f_max post-PEX | 72.5 MHz | 54.4 MHz |
| Hold violations | — | **0** |
| DRC / LVS | — | **clean / pass** |

Area growth is driven almost entirely by the wider, deeper SRAM macros; the fault-tolerance logic
itself is nearly free (repair buffers: 0.5% each). The power increase is concentrated in the clock
network (6.89 → 10.20 mW) from the CTS and buffering needed around the relocated macros — the
sequential and combinational logic actually got *cheaper*.

Backend work that moved the needle:
- **Floorplan:** macros at opposite vertical extremes, rotated so signal pins face the core logic,
  shifted toward the clock pin — minimizing wirelength under a severely macro-constrained core.
- **Placement:** density swept 55–70%, with 65% minimizing DRVs; `set_placement_padding` used to
  reach a clean DRC report. **Fanout violations cut from 122 → 5**, better than the reference flow.
- **CTS:** balanced clock levels and strict sink clustering spread buffers evenly, which also
  reduced dynamic IR drop as a side effect.
- **Power grid:** retuned M4 / TopMetal1 pitch and width to land static IR drop at 1.19% (VDD) and
  1.07% (VSS) — comfortably under the 2% industry threshold — while freeing routing tracks for
  congestion relief.

<p align="center">
  <img src="doc/01_croc.floorplan.png" width="32%">
  <img src="doc/03_CTS.png" width="32%">
  <img src="doc/07_IRdrop.png" width="32%">
</p>
<p align="center"><i>Final floorplan · clock tree · IR-drop distribution</i></p>

---

## Repository map

| Path | Contents |
|---|---|
| [`rtl/user_domain/secded_byte.sv`](rtl/user_domain/secded_byte.sv) | Hsiao (13,8) byte encoder & decoder |
| [`rtl/user_domain/secded_sram_impl.sv`](rtl/user_domain/secded_sram_impl.sv) | Drop-in protected SRAM wrapper |
| [`rtl/user_domain/secded_repair_buffer.sv`](rtl/user_domain/secded_repair_buffer.sv) | Silent writeback / scrubbing logic |
| [`rtl/user_pkg.sv`](rtl/user_pkg.sv) | `SECDEDBypass` feature toggle |
| [`rtl/test/`](rtl/test/) | `tb_secded_byte`, `tb_fault_injection_croc`, `tb_sec_repair`, benches |
| [`sw/`](sw/) | `bitflip_verify.c`, `sec_repair.c`, `secded_bench.c`, `hw_secded_bench.c` |
| [`yosys/`](yosys/) · [`openroad/`](openroad/) · [`klayout/`](klayout/) · [`calibre/`](calibre/) | Synthesis, P&R, DRC, LVS flows |
| [`doc/`](doc/) | Report and figures |

Built on the [Croc SoC](https://github.com/pulp-platform/croc) (PULP Platform, ETH Zürich / Uni Bologna).
