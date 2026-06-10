# asmgen — Roadmap

go-asmgen grows along two axes: **wider type support within an architecture**,
and **more architectures** reusing the same emit layer.

## v0 — proof of pipeline (here)

- arm64, ABI0, sequences of 8-byte int/ptr arguments and results.
- ISA-agnostic `internal/emit` text writer.
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

## Shared ABI0 model + riscv64 — done

- Extracted the architecture-independent ABI0 layout into `internal/abi`; arm64
  re-exports it with no behaviour change.
- Added **riscv64** as a thin second architecture over the shared model — only a
  move table differs (`MOV` for 8-byte ints, `MOVF`/`MOVD` for floats). 100%
  coverage, asmdecl + `cmd/asm` validated, and runtime-proven under qemu-user.
  See [Second architecture: riscv64](riscv64.md).

## Aggregates and vectors

- Struct and array arguments/results (field decomposition).
- Vector (`V`-register) values.

## More architectures

- **loong64**, reusing the shared layout model (same recipe as riscv64).

## Possible future work

- Derive instruction-mnemonic tables from `cmd/internal/obj` to catch typos at
  generation time — still delegating actual encoding to `cmd/asm`.
- Differential encoding checks wired into CI as an additional guard against
  emit-layer regressions.
