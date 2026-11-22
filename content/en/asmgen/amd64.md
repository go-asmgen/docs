---
title: "Fourth target: amd64"
weight: 90
---

# Fourth target: amd64 {#fourth-target-amd64}

amd64 is already well served by [avo][avo], which encodes instructions at the
byte level and models the instruction set in depth. go-asmgen supports amd64 not
to replace that, but so a **single uniform builder** spans every 64-bit Go
target — the same code shape generates arm64, riscv64, loong64, ppc64le, s390x,
and amd64.

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

## CPU feature probes {#feature-probes}

`amd64.FeatureProbe(name, feature)` emits a NOSPLIT `func() bool` that reports
whether both the CPU and the OS support a feature, so a package that dispatches
SIMD kernels needs no runtime dependency to gate them. The OS half is checked
too: OSXSAVE, then XCR0 for the register state the instructions touch (XMM and
YMM for VEX forms, plus opmask and ZMM for EVEX.512).

| Feature | CPUID bit | Also requires |
| --- | --- | --- |
| `AVX2` | leaf 7, EBX 5 | AVX, YMM state |
| `AVX512F` | leaf 7, EBX 16 | opmask and ZMM state |
| `AVX512VNNI` | leaf 7, ECX 11 | AVX512F (512-bit form) |
| `AVX512VPOPCNTDQ` | leaf 7, ECX 14 | AVX512F |
| `VPCLMULQDQ` | leaf 7, ECX 10 | AVX512F: the **512-bit** form |
| `VPCLMULQDQ256` | leaf 7, ECX 10 | AVX, YMM state, **no AVX512F** (Zen 3, Alder Lake) |
| `AVXVNNI` | leaf 7 sub-leaf 1, EAX 4 | AVX, YMM state; VEX-encoded, no AVX-512 |
| `F16C` | leaf 1, ECX 29 | AVX, YMM state |
| `BMI2` | leaf 7, EBX 8 | nothing (general-purpose registers) |

Gate a YMM CRC or GHASH fold on `VPCLMULQDQ256`, and a ZMM one on `VPCLMULQDQ`.
`AVX512VNNI` and `AVXVNNI` are distinct bits: one does not imply the other.
`BMI2` says PDEP and PEXT exist, not that they are fast (they are microcoded
before AMD Zen 3). The bits were cross-checked against
`golang.org/x/sys/cpu`. Each probe's emitted sequence is pinned by a test; none
has been run on hardware that lacks the feature. The AVX-512 probes answer false
on macOS, where Darwin enables ZMM state lazily.

## Validation {#validation}

amd64 is the easiest target to runtime-test: the GitHub-hosted ubuntu runner is
amd64, so the generated assembly runs natively — no emulation. Locally on Apple
Silicon it runs under Rosetta (`GOARCH=amd64 go test`). The asmgen CI has a
dedicated native `asm-amd64` job covering the scalar and SSE-SIMD examples.

See also [SIMD]({{< relref "simd.md" >}}) for the SSE packed-add example.

[avo]: https://github.com/mmcloughlin/avo
