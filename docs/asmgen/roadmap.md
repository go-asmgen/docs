# asmgen — Roadmap

go-asmgen grows along two axes: **wider type support within an architecture**,
and **more architectures** reusing the same emit layer.

## v0 — proof of pipeline (here)

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

## Shared ABI0 model + riscv64 + loong64 — done

- Extracted the architecture-independent ABI0 layout into `abi`; arm64
  re-exports it with no behaviour change.
- Added **riscv64** ([page](riscv64.md)) and **loong64** ([page](loong64.md)) as
  thin architectures over the shared model — each only a move table (`MOV`/`MOVV`
  for 8-byte ints, `MOVF`/`MOVD` for floats). 100% coverage, asmdecl + `cmd/asm`
  validated, and runtime-proven under qemu-user.

## Aggregates — done

- Struct, slice, and string parameters laid out by Go's struct rules, each field
  addressed as `name_field+offset(FP)`, asmdecl- and runtime-validated. See
  [Aggregates](aggregates.md).

## Arrays — done

- Fixed-size `[n]T` arrays passed by value, addressed element-wise
  (`name_0 … name_(n-1)`) via `abi.Array`; `Signature.Slot` for offset lookup.
  asmdecl-clean and runtime-proven. See [Aggregates](aggregates.md#fixed-size-arrays).

## Vectors

- SIMD code generation works today through `Raw` over **pointer** arguments
  (SSE/NEON) — see [SIMD](simd.md). A *single* vector load of a whole by-value
  array is not asmdecl-clean, so passing vectors by value is not pursued.
- Possible future work: a typed vector-load helper that emits the
  pointer-based load/op/store sequence, removing the `Raw` boilerplate.

## More architectures

- Any further 64-bit target (e.g. a future port) follows the same recipe:
  re-export the shared `abi` types and add one `loadMnemonic`/`storeMnemonic`
  pair.

## Possible future work

- Derive instruction-mnemonic tables from `cmd/internal/obj` to catch typos at
  generation time — still delegating actual encoding to `cmd/asm`.
- Differential encoding checks wired into CI as an additional guard against
  emit-layer regressions.
