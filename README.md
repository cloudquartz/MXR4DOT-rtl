# MXDOTP coprocessor for CV32E40X

This repository adds a CV-X-IF (eXtension interface) coprocessor for
microscaling (MX) dot products to the OpenHW CV32E40X RISC-V core. The
coprocessor accepts custom instructions on the custom-0 opcode
(`7'b0001011`). The core has one small modification: EX is halted while a
load/store in EX sits behind an offloaded instruction in WB
(`xif_lsu_stall`). It is computed in `rtl/cv32e40x_controller_bypass.sv`,
declared in `rtl/include/cv32e40x_pkg.sv` (`ctrl_byp_t`), and ORed into
`halt_ex` in `rtl/cv32e40x_controller_fsm.sv`. The coprocessor implements
four number formats, each with its own engine:

| Format | Engine (`rtl/mxdotp/`) |
|---|---|
| MXFP4 (E2M1) | `mxdotp_fp4_fused_engine.sv` |
| MXFP8 (E4M3 / E5M2) | `mxdotp_fp8_fused_engine.sv` |
| M2XFP4 | `mxdotp_m2xfp4_fused_engine.sv` |
| MXFP4R (residue pair) | `mxdotp_dotp_engine.sv` (MXDOTP: raw dot product) + `mxdotp_final_engine.sv` (MXFINAL: apply shared scales, accumulate into FP32, round once), connected by `mxdotp_mailbox.sv` |

`mxdotp_xif.sv` is the CV-X-IF front end that decodes the instructions and
routes them to the engines. `mxdotp_core_top.sv` connects the core and the
coprocessor. Instruction encodings and the per-format accumulator frames are
documented in `mxdotp_pkg.sv`.

The upstream CV32E40X README is kept as `README_CV32E40X.md`.

Accuracy-evaluation code and results are in a companion repository: https://github.com/cloudquartz/MXR4DOT-eval.

## Repository layout

| Path | Contents |
|---|---|
| `rtl/` | CV32E40X core RTL (upstream, plus that one change) |
| `rtl/mxdotp/` | The coprocessor: CV-X-IF front end, decoder, engines, mailbox, and shared `mx_*.sv` datapath blocks |
| `bhv/`, `sva/` | Upstream core behavioural wrapper, RVFI, and SVA checkers |
| `tb/` | Verilator testbenches: full system (`tb_mxdotp_core.sv`) and one unit testbench per engine (`tb_*_unit.sv`); `Makefile` |
| `tb/scripts/` | Helper scripts for stream, pipeline, and replay analysis |
| `tb/pair_replay/` | Replay checks for the MXFP4R pair |
| `sw/` | Bare-metal RISC-V kernels per format (`mxfp4`, `mxfp8`, `mxfp4r`, `m2xfp4`), plus `smoke`, `q10_isolation`, and shared startup code in `common/` |
| `verification/` | Python golden models for each engine (`*_golden.py`) and residue cross-checks |
| `syn/` | Synthesis: wrappers, filelists, constraints, Cadence scripts (see below) |
| `analysis/` | `bw_model.py` (memory-traffic model), `regime_a_runs.sh` (runs the K=4096 loop programs and feeds `bw_model.py`), and `configs/` (model configs used by `bw_model.py`, with `snapshot_hashes.txt`) |
| `constraints/` | Upstream core timing constraints |
| `docs/` | Upstream CV32E40X user manual |
| `reference/` | Upstream third-party reference code |

## Running the testbenches

Requirements: Verilator 5 (`verilator5` or `verilator` on `PATH`; the builds
use `--timing`) and Python 3 (the unit tests generate their vectors from
the golden models in `verification/`).

From the repository root:

    make -C tb units    # every engine unit test against its golden model
    make -C tb full     # full-system regression: core + coprocessor (tb_mxdotp_core)

Single unit tests: `make -C tb fp4`, `fp8`, `m2`, `mxdotp`, `mxfinal`.
`make -C tb clean` removes all build output (`tb/build/`). Software-kernel
runs (`make -C tb loop PROG=<kernel>`) also need a RISC-V GCC toolchain with
the prefix `riscv64-unknown-elf-`.

## Synthesis

Everything for synthesis is under `syn/`:

- `syn/wrappers/mx_syn_<design>.sv`: synthesis-only top levels (not design
  RTL), one per design: `fp4`, `fp8`, `m2`, `dotp`, `mxfinal`, `pair` (the
  MXFP4R pair with its mailbox), `xif` (the whole coprocessor), `eth`.
- `syn/filelists/<design>.f`: source list per design, paths relative to the
  repository root.
- `syn/constraints/mx.sdc`: timing constraints.
- `syn/cadence/`: Genus/Innovus scripts. `run.sh <design> <period_ns>
  [prep|syn|pnr|all]` runs one design; `sweep.sh` runs a sweep. The tool
  versions and technology are given in the header of `run.sh`.

## ETH MXDOTP RTL not included

The ETH MXDOTP RTL is used only as a synthesis baseline (`eth` design) and
is not part of any simulation build. It is not included here for licensing
reasons. To synthesise the baseline, obtain it from its upstream and place
the files named in `syn/filelists/eth.f` in `syn/third_party/eth_mxdotp/`.
