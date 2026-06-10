# asmgen — Roadmap

go-asmgen grows along two axes: **wider type support within an architecture**,
and **more architectures** reusing the same emit layer.

## v0 — proof of pipeline (here)

- arm64, ABI0, sequences of 8-byte int/ptr arguments and results.
- ISA-agnostic `internal/emit` text writer.
- End-to-end `go:generate` example, validated by asmdecl + a native arm64
  runtime test, with the library held to 100% coverage.

## Widen arm64 type support

- 1/2/4-byte integers, with correct move selection
  (`MOVB`/`MOVH`/`MOVW`/`MOVD`).
- Floating-point registers (`F0..`) and `FMOV*` moves.
- Proper alignment and padding rules for mixed-size and struct arguments.
- Each case tested against `go vet` asmdecl.

## More architectures

- **riscv64** (starting from RV64GC), reusing the emit layer.
- **loong64**.

## Possible future work

- Derive instruction-mnemonic tables from `cmd/internal/obj` to catch typos at
  generation time — still delegating actual encoding to `cmd/asm`.
- Differential encoding checks wired into CI as an additional guard against
  emit-layer regressions.
