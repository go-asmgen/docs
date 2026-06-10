# Fourth target: amd64

amd64 is already well served by [avo][avo], which encodes instructions at the
byte level and models the instruction set in depth. go-asmgen supports amd64 not
to replace that, but so a **single uniform builder** spans every 64-bit Go
target — the same code shape generates arm64, riscv64, loong64, and amd64.

It reuses the shared ABI0 layout unchanged. amd64's move table is a little richer
than the RISC targets', in two ways:

- **Sub-word loads use explicit sign/zero-extending mnemonics.** Where arm64 has
  `MOVB`/`MOVBU`, amd64 has `MOVBQSX` (sign-extend byte→quad) / `MOVBQZX`
  (zero-extend), and likewise `MOVWQSX`/`MOVWQZX` and `MOVLQSX`/`MOVLQZX`.
- **Floats use the SSE scalar moves** `MOVSS` (single) / `MOVSD` (double) on the
  X registers, rather than a dedicated float move like arm64's `FMOVS`/`FMOVD`.

| Type | arm64 | amd64 |
| --- | --- | --- |
| `int64` / `uint64` / pointer | `MOVD` | `MOVQ` |
| `int8` (load, sign) | `MOVB` | `MOVBQSX` |
| `uint8` (load, zero) | `MOVBU` | `MOVBQZX` |
| `int32` (load, sign) | `MOVW`… | `MOVLQSX` |
| store `int8`/`int16`/`int32` | `MOVB`/`MOVH`/`MOVW` | `MOVB`/`MOVW`/`MOVL` |
| `float32` / `float64` | `FMOVS` / `FMOVD` | `MOVSS` / `MOVSD` |

amd64 arithmetic is two-operand (`ADDQ b, a` computes `a += b`), so the example
generator accumulates into the first register. The ABI0 offsets are identical to
the other targets.

## Validation

amd64 is the easiest target to runtime-test: the GitHub-hosted ubuntu runner is
amd64, so the generated assembly runs natively — no emulation. Locally on Apple
Silicon it runs under Rosetta (`GOARCH=amd64 go test`). The asmgen CI has a
dedicated native `asm-amd64` job covering the scalar and SSE-SIMD examples.

See also [SIMD](simd.md) for the SSE packed-add example.

[avo]: https://github.com/mmcloughlin/avo
