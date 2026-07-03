# wasm — Overview

[`go-asmgen/wasm`](https://github.com/go-asmgen/wasm) is a WebAssembly text
(`.wat`) emitter modeled on
[`go-asmgen/emit`](https://github.com/go-asmgen/asmgen/tree/main/emit). Where
the main `asmgen` module builds Plan 9 assembly kernels for the six 64-bit Go
targets (amd64 / arm64 / ppc64le / riscv64 / loong64 / s390x), this module
covers **the seventh target: wasm-SIMD (v128)** — the module text a Go host
consumes via `//go:wasmimport`.

## Why wasm needs a separate emitter

The other six architectures share Plan 9 assembly and the ABI0 model. wasm has
neither:

- Go's compiler **does not emit `v128`** from Go source, so wasm-SIMD kernels
  have to be external.
- Plan 9 assembly has **no wasm dialect** — the toolchain assembler
  (`cmd/asm`) does not compile WAT.
- The Go wasm ABI is **stack-based**, not register-based: locals + a value
  stack, not the `FP`/`SP`/register-file model asmgen's other builders share.

Everything else transfers, though. The same layering pattern the `asmgen`
module uses — a per-target emit surface over a shared abstract model —
produces well-formed WAT programmatically, byte-equivalent to a hand-authored
kernel and dropped straight into a `//go:wasmimport` consumer.

## What the module ships

A single package, `github.com/go-asmgen/wasm`, that exposes:

- `Module` — a whole WAT `(module ...)` with imports and functions.
- `Function` — one `(func $name (export "name") ...)` with params, results,
  locals, and a stack-op body.
- Value types: `i32`, `i64`, `f32`, `f64`, `v128`.
- Every v128 op the five shipped kernels need: memory ops, boolean, i8x16
  compare/arith/splat/popcnt, horizontal aggregation (bitmask), shuffle/
  swizzle, widening reductions (extadd_pairwise), and enough scalar plumbing
  for the surrounding control flow.

Eight kernels, each with 100% statement coverage, a golden-file test, and
a wazero end-to-end cross-check against a Go reference:

| Kernel | Reference | Wasm-SIMD ops it showcases |
|---|---|---|
| `matchlen` | `bytes.Equal` | `v128.load`, `i8x16.eq`, `i8x16.all_true` |
| `hex` | `encoding/hex` | `i8x16.shr_u`, `i8x16.swizzle`, `i8x16.shuffle` |
| `hex_decode` | `encoding/hex.DecodeString` | 3× `i8x16.ge_s`/`le_s`, `i8x16.shl`, `v128.store64_lane` |
| `popcount` | `math/bits` | `i8x16.popcnt`, `i16x8.extadd_pairwise_i8x16_u`, `i32x4.extadd_pairwise_i16x8_u` |
| `toupper` | `strings.ToUpper` (ASCII) | `i8x16.ge_s`, `i8x16.le_s`, `i8x16.sub` |
| `memchr` | `bytes.IndexByte` | `i8x16.splat`, `i8x16.bitmask`, `i32.ctz` |
| `isascii` | ASCII-only preflight | `i8x16.bitmask` (MSB-of-lane as "is >= 128") |
| `utf8len` | `utf8.RuneCount` (valid UTF-8) | `i8x16.bitmask` + scalar `i32.popcnt` reducer |

## What the module does not do

- **Only WAT (text) output.** `.wasm` binary encoding is `wat2wasm`'s job
  (part of [WABT](https://github.com/WebAssembly/wabt)).
- **Only wasm-SIMD (v128).** No relaxed-SIMD, no multi-memory.
- **Does not touch Go binding.** That's a consumer-side concern; see
  [`go-simd/matchlen-wasm`](https://github.com/go-simd/matchlen-wasm) for the
  `//go:wasmimport` wrapper pattern.

## Where to next

- **[Quick start](quickstart.md)** — generate a kernel, compile it, verify it
  under wazero.
- **[Kernels](kernels.md)** — the five shipped kernels with signatures and
  emit-surface excerpts.
- **[Design notes](design.md)** — how the wasm module maps to the asmgen
  layering, what changed, what stayed the same.
- **[Roadmap](roadmap.md)** — utf-8 validation, base64, adler32, folding into
  the main `asmgen` module.
