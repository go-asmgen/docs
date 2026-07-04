# wasm — Overview

[`go-asmgen/asmgen/wasm`](https://github.com/go-asmgen/asmgen/tree/main/wasm)
is a WebAssembly text (`.wat`) emitter modeled on
[`emit`](https://github.com/go-asmgen/asmgen/tree/main/emit), shipped alongside
the six existing arch packages (amd64 / arm64 / ppc64le / riscv64 / loong64 /
s390x). It covers **the seventh target: wasm-SIMD (v128)** — the module text
a Go host consumes via `//go:wasmimport`.

!!! note "Was previously a sibling module"
    The wasm surface used to ship as a standalone `go-asmgen/wasm`
    module. It was folded into the main `go-asmgen/asmgen` repo as a
    peer package; the standalone repo has since been retired. Pin
    `github.com/go-asmgen/asmgen/wasm/...` and
    `github.com/go-asmgen/asmgen/examples/wasm/<kernel>`.

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

A single package, `github.com/go-asmgen/asmgen/wasm`, that exposes:

- `Module` — a whole WAT `(module ...)` with imports and functions.
- `Function` — one `(func $name (export "name") ...)` with params, results,
  locals, and a stack-op body.
- Value types: `i32`, `i64`, `f32`, `f64`, `v128`.
- Every v128 op the five shipped kernels need: memory ops, boolean, i8x16
  compare/arith/splat/popcnt, horizontal aggregation (bitmask), shuffle/
  swizzle, widening reductions (extadd_pairwise), and enough scalar plumbing
  for the surrounding control flow.

Thirteen kernels, each with 100% statement coverage, a golden-file test,
and a wazero end-to-end cross-check against a Go reference:

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
| `json_clean` | JSON-string bulk-copy preflight | 2× `i8x16.eq` + `v128.and`/zero-eq (avoids signed-le_s trap) |
| `adler32` | `hash/adler32` | `i16x8.extmul_low/high_i8x16_u` for the weighted-byte sum |
| `base64_encode` | `encoding/base64.StdEncoding` | Lemire's PMULHUW emulated via `i32x4.extmul + shr_u + i16x8.narrow` |
| `indexany4` | `bytes.IndexAny` (fixed 4-needle set) | 4× `i8x16.eq` + `v128.or` + `i8x16.bitmask` + `i32.ctz` |
| `base64_decode` | `encoding/base64.StdEncoding` (decode) | 5-range mask+sub + 3× `i8x16.shl` + 2× `i8x16.shr_u` + 4 shuffle merges |

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
- **[Kernels](kernels.md)** — the nine shipped kernels with signatures and
  emit-surface excerpts.
- **[Design notes](design.md)** — how the wasm module maps to the asmgen
  layering, what changed, what stayed the same.
- **[Consumer CI](consumer-ci.md)** — drift + bench + arch-native CI pattern
  used by `matchlen-wasm`, end-to-end validated.
- **[Roadmap](roadmap.md)** — utf-8 full validation, base64, adler32,
  folding into the main `asmgen` module.
