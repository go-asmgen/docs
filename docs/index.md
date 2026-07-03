# go-asmgen documentation

**Ergonomic generation of Go-compatible Plan 9 assembly for every 64-bit Go
target** — **amd64**, **arm64**, **riscv64**, **loong64**, **ppc64le** (VSX),
and **s390x** (vector facility, big-endian).

[avo][avo] does this for amd64 by encoding instruction bytes itself, which is
exactly what makes extending it to new ISAs expensive. go-asmgen instead **emits
Plan 9 assembly text and lets the Go toolchain assembler (`cmd/asm`) encode it**
— so each architecture is only a thin move/register surface over a **shared**
ABI0 layout model, not a byte-level encoder. avo remains the richer choice for
amd64-specific work; go-asmgen offers one uniform builder across every target.

| Package | What it is |
| --- | --- |
| [`amd64`](asmgen/amd64.md) | builder emitting Plan 9 amd64 instructions (`MOVQ`, `MOVSS`/`MOVSD`) |
| [`arm64`](asmgen/index.md) | builder emitting Plan 9 arm64 instructions (`MOVD`, `FMOVS`/`FMOVD`) |
| [`riscv64`](asmgen/riscv64.md) | builder emitting Plan 9 riscv64 instructions (`MOV`, `MOVF`/`MOVD`) |
| [`loong64`](asmgen/loong64.md) | builder emitting Plan 9 loong64 instructions (`MOVV`, `MOVF`/`MOVD`) |
| [`ppc64le`](asmgen/ppc64le.md) | builder emitting Plan 9 ppc64le instructions (`MOVD`, `FMOVS`/`FMOVD`) |
| [`s390x`](asmgen/s390x.md) | builder emitting Plan 9 s390x instructions (`MOVD`, `FMOVS`/`FMOVD`; big-endian) |
| `abi` | the architecture-independent ABI0 layout model all builders share |
| `emit` | a deliberately dumb, ISA-agnostic writer of well-formed Plan 9 `.s` text |

Start with the [Quick start](asmgen/quickstart.md), read the
[ABI0 & design notes](asmgen/design.md) for why it is built this way, see
[Aggregates](asmgen/aggregates.md) for struct/slice/string parameters, then
[riscv64](asmgen/riscv64.md), [loong64](asmgen/loong64.md),
[ppc64le](asmgen/ppc64le.md) and [s390x](asmgen/s390x.md) for how cheaply a
new ISA drops in.

## The seventh target: wasm-SIMD

The same layering pattern runs one more target — wasm-SIMD (v128) — in a
sibling module [`go-asmgen/wasm`](https://github.com/go-asmgen/wasm). Go's
compiler does not emit `v128` from Go source, and Plan 9 assembly has no
wasm dialect, so wasm-SIMD is necessarily an external-kernel story. The
emitter builds WAT text programmatically, `wat2wasm` compiles it, and the
Go host imports it via `//go:wasmimport`.

v0.1.0 ships five kernels — matchlen, hex, popcount, toupper, memchr —
each byte-equivalent to a hand-written reference and wazero-verified against
a Go stdlib function. Start with the [wasm overview](wasm/index.md) or the
[quick start](wasm/quickstart.md).

Source lives at [github.com/go-asmgen/asmgen](https://github.com/go-asmgen/asmgen)
and [github.com/go-asmgen/wasm](https://github.com/go-asmgen/wasm).

[avo]: https://github.com/mmcloughlin/avo
