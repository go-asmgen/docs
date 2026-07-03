# wasm — Roadmap

Where the WAT emitter sits today and what is planned. The order below is
priority, not commitment.

## Grow the kernel catalogue

The kernels shipped in v0.1.0 (matchlen, hex, popcount, toupper, memchr)
cover the most common wasm-consumer patterns: byte-compare, ASCII encode,
bit-count, ASCII case-fold, byte-search. Next up:

- **utf-8 validation** — the classic wasm-SIMD showcase. Lemire &
  Keiser's algorithm uses `v128.const` LUTs, `i8x16.swizzle` for range
  lookup, and `i8x16.shr_u` for continuation-byte checks. The kernel
  exercises the shuffle/LUT surface aggressively.
- **base64 encode** — Lemire's SSE algorithm ported. Needs `i16x8.mul` and
  its friends for the 3-byte-to-4-char index-extraction step; adds
  saturating i16x8 arithmetic to the emit surface.
- **adler32 / crc32** — checksum families used everywhere in zlib-family
  formats. `i16x8.mul` again, plus i32x4 multiply-accumulate patterns.
- **`bytes.IndexAny`** — multi-needle memchr. Extends memchr with a small
  needle-set LUT and a per-lane compare-any pattern.

Adding a new kernel is a `main.go` in a new subdirectory plus a
`main_test.go` with the golden-file pattern; adding a new op is one line in
`emit.go`. Nothing structural changes.

## ~~Fold into the main asmgen module~~ — done

The wasm surface used to ship as a stand-alone `go-asmgen/wasm` module.
As of `go-asmgen/asmgen@v0.6.0` it lives inside the main asmgen module
as a peer of `amd64` / `arm64` / …:

- [`wasm/`](https://github.com/go-asmgen/asmgen/tree/main/wasm) —
  the emit surface.
- [`examples/wasm/`](https://github.com/go-asmgen/asmgen/tree/main/examples/wasm)
  — one subdirectory per shipped kernel, each with the `go:generate`
  line and the committed `.wat.golden`.
- A `wasm-e2e` job in the top-level `ci.yml` regenerates every kernel,
  drift-gates against the committed golden, `wat2wasm`-compiles, and
  runs the wazero verifier — one job for all nine kernels because
  wasm-SIMD is arch-agnostic.

A future extension to the shared `abi` package would lower Go slice
headers (`base, len, cap`) into wasm `(i32, i32, i32)` param triples
for `//go:wasmimport` consumers automatically. Not blocking anything
today — every shipped kernel hand-writes the ABI at its call sites.

## Grow the consumer ecosystem

Each shipped kernel deserves a `go-simd/*-wasm` consumer alongside the
existing [`go-simd/matchlen-wasm`](https://github.com/go-simd/matchlen-wasm):

- `go-simd/hex-wasm`
- `go-simd/popcount-wasm`
- `go-simd/toupper-wasm`
- `go-simd/memchr-wasm`

Every consumer follows the same shape as matchlen-wasm: a `//go:generate`
directive pinning the emitter version, `wasm-drift.yml` for the drift gate,
`wasm-bench.yml` for the regression gate, and a small Go wrapper that fronts
the `//go:wasmimport`.

## Non-goals

- **wasm-binary encoding.** WAT text plus `wat2wasm` is enough; the module
  will not grow a binary encoder.
- **Non-SIMD wasm.** The point of the module is v128; a scalar-only wasm
  target is already served by Go's stock wasm output.
- **Runtime selection.** The kernel signatures target the SIMD proposal
  unconditionally. Consumers that need a scalar fallback should ship one in
  Go alongside the wasmimport call — that's a consumer concern, not an
  emitter concern.
