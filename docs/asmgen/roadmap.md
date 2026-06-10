# asmgen — Roadmap

go-asmgen grew along two axes: **wider type support within an architecture**,
and **more architectures** over a shared ABI0 layout model. Both are now broadly
covered; the library is released (latest: **v0.2.0**).

## v0 — proof of pipeline — done

- arm64, ABI0, sequences of 8-byte int/ptr arguments and results.
- ISA-agnostic `emit` text writer.
- End-to-end `go:generate` example, validated by asmdecl + a native arm64
  runtime test, with the library held to 100% coverage.

## Widen arm64 scalar support — done

- 1/2/4/8-byte signed and unsigned integers and pointers, with correct move
  selection (`MOVB`/`MOVBU`, `MOVH`/`MOVHU`, `MOVW`/`MOVWU`, `MOVD`) and sub-word
  sign/zero extension.
- Floating-point registers (`F0..`) with `FMOVS`/`FMOVD`.
- Word-aligned result area and per-type argument alignment.
- Every case validated against `go vet` asmdecl and runtime tests on native
  arm64 (see `examples/types`).

## Shared ABI0 model + more architectures — done

- Extracted the architecture-independent ABI0 layout into `abi`; arm64
  re-exports it with no behaviour change.
- Added **riscv64** ([page](riscv64.md)), **loong64** ([page](loong64.md)), and
  **amd64** ([page](amd64.md)) as thin architectures over the shared model — each
  only a move table. 100% coverage, asmdecl + `cmd/asm` validated, and
  runtime-proven (natively on amd64/arm64, under qemu-user for riscv64/loong64).
- Promoted `abi` and `emit` out of `internal/` so the library is importable.

## Aggregates — done

- Struct, slice, and string parameters laid out by Go's struct rules, each field
  addressed as `name_field+offset(FP)`, asmdecl- and runtime-validated. See
  [Aggregates](aggregates.md).

## Arrays — done

- Fixed-size `[n]T` arrays passed by value, addressed element-wise
  (`name_0 … name_(n-1)`) via `abi.Array`; `Signature.Slot` for offset lookup.
  asmdecl-clean and runtime-proven. See [Aggregates](aggregates.md#fixed-size-arrays).

## SIMD — done (via `Raw`)

- Packed-add examples on **all four** targets, runtime-proven, including the
  256-bit widths: SSE2 + **AVX2** (amd64), NEON (arm64), RVV (riscv64), LSX +
  **LASX** (loong64). Emitted through `Raw` over **pointer** arguments — see
  [SIMD](simd.md). A *single* vector load of a whole by-value array is not
  asmdecl-clean, so passing vectors by value is not pursued.
- Possible future work: a typed vector-load helper that emits the
  pointer-based load/op/store sequence, removing the `Raw` boilerplate.

## Stack frames and TEXT flags — done

- `NewFuncFlags` emits any Plan 9 TEXT flags (`NOSPLIT`, `NOSPLIT|NOFRAME`, or
  none); `frameSize > 0` reserves stack locals addressed `name-N(SP)` via `Raw`.
  See [TEXT flags and stack frames](design.md#text-flags-and-stack-frames) and
  `examples/frame` — a non-`NOSPLIT` function that spills to a 16-byte frame,
  runtime-proven.

## Adding another architecture

- Any further 64-bit target follows the same recipe: re-export the shared `abi`
  types and add one `loadMnemonic`/`storeMnemonic` pair.

## Possible future work

- Derive instruction-mnemonic tables from `cmd/internal/obj` to catch typos at
  generation time — still delegating actual encoding to `cmd/asm`.
- Differential encoding checks wired into CI as an additional guard against
  emit-layer regressions.
