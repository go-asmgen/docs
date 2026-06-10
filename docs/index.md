# go-asmgen documentation

**Ergonomic generation of Go-compatible Plan 9 assembly for non-amd64
architectures** — **arm64**, **riscv64**, and **loong64**.

go-asmgen is the multi-architecture counterpart to what [avo][avo] does for
amd64. avo encodes instruction bytes itself, which is exactly what makes
extending it to new ISAs expensive. go-asmgen instead **emits Plan 9 assembly
text and lets the Go toolchain assembler (`cmd/asm`) encode it** — so each new
architecture needs only a thin move/register surface over a **shared** ABI0
layout model, not a byte-level encoder.

| Package | What it is |
| --- | --- |
| [`arm64`](asmgen/index.md) | builder emitting Plan 9 arm64 instructions (`MOVD`, `FMOVS`/`FMOVD`) |
| [`riscv64`](asmgen/riscv64.md) | builder emitting Plan 9 riscv64 instructions (`MOV`, `MOVF`/`MOVD`) |
| [`loong64`](asmgen/loong64.md) | builder emitting Plan 9 loong64 instructions (`MOVV`, `MOVF`/`MOVD`) |
| `internal/abi` | the architecture-independent ABI0 layout model all builders share |
| `internal/emit` | a deliberately dumb, ISA-agnostic writer of well-formed Plan 9 `.s` text |

Start with the [Quick start](asmgen/quickstart.md), read the
[ABI0 & design notes](asmgen/design.md) for why it is built this way, see
[Aggregates](asmgen/aggregates.md) for struct/slice/string parameters, then
[riscv64](asmgen/riscv64.md) and [loong64](asmgen/loong64.md) for how cheaply a
new ISA drops in.

Source lives at [github.com/go-asmgen/asmgen](https://github.com/go-asmgen/asmgen).

[avo]: https://github.com/mmcloughlin/avo
