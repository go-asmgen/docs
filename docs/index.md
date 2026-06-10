# go-asmgen documentation

**Ergonomic generation of Go-compatible Plan 9 assembly for non-amd64
architectures**, starting with **arm64**.

go-asmgen is the multi-architecture counterpart to what [avo][avo] does for
amd64. avo encodes instruction bytes itself, which is exactly what makes
extending it to new ISAs expensive. go-asmgen instead **emits Plan 9 assembly
text and lets the Go toolchain assembler (`cmd/asm`) encode it** — so each new
architecture needs only an ABI layout model plus a thin instruction-emit
surface, not a byte-level encoder.

| Package | What it is |
| --- | --- |
| [`arm64`](asmgen/index.md) | ABI0 frame-layout model + an ergonomic builder that emits Plan 9 arm64 instructions |
| `internal/emit` | a deliberately dumb, ISA-agnostic writer of well-formed Plan 9 `.s` text |

Start with the [Quick start](asmgen/quickstart.md), then read the
[ABI0 & design notes](asmgen/design.md) for why it is built this way and what
is and is not correct in v0.

Source lives at [github.com/go-asmgen/asmgen](https://github.com/go-asmgen/asmgen).

[avo]: https://github.com/mmcloughlin/avo
