---
title: "Instructions cmd/asm lacks"
weight: 120
---

# Instructions cmd/asm lacks {#instructions-cmdasm-lacks}

go-asmgen emits Plan 9 assembly and lets the Go assembler, `cmd/asm`, encode it.
Sometimes a kernel needs an instruction `cmd/asm` cannot assemble yet. go-asmgen
then emits it as a `WORD` for the time being, and keeps everything needed to
remove that `WORD` later.

## What gets emitted {#what-gets-emitted}

```
WORD $0xf001130f // XVMADDADP VS33, VS34, VS32
```

The encoded word comes first. The comment is the spelling proposed for `cmd/asm`,
with operands in its order, so the line already shows what it will become.

As of v0.15.0:

| Architecture | Instructions | Emitted as |
|---|---|---|
| arm64 | `VFADD2D` `VFSUB2D` `VFMUL2D` `VFNEG2D` `VFMLA2D` `VFMLS2D` | mnemonics (Go 1.27) |
| arm64 | `VFMLAElem2D` `VFMLSElem2D` `VFMULElem2D` `VFMLAElem4S` `VFMLSElem4S` `VFMULElem4S` (by element, from v0.22.0) | `WORD` |
| loong64 | `VLDREPLD` `XVLDREPLD` | `VMOVQ off(R), V.V2` / `XVMOVQ off(R), X.V4`, except offset −2048 |
| loong64 | `VFMADDD` `XVFMADDD` | `WORD` |
| loong64 | `VFMADDF` `XVFMADDF` (float32 fused multiply-add, from v0.23.0) | `WORD` |
| loong64 | `CSRRD` `CSRWR` `CSRXCHG` (from v0.21.0) | `WORD` |
| ppc64le | `XVADDDP` `XVSUBDP` `XVMULDP` `XVDIVDP` `XVMADDADP` `XVMAXDP` `XVMINDP` `XVSQRTDP` | `WORD` |

## The registry {#the-registry}

Each `WORD` comes from an entry in `internal/gap`, which holds:

- the encoder;
- reference encodings from independent assemblers (GNU as on the real
  hardware; llvm-mc agrees on all 102 cases), covering every register field at
  its ends;
- the opcode mask;
- the proposed Plan 9 spelling.

The architecture packages only call into it, so there is one source.

The loong64 privileged CSR access instructions `csrrd`, `csrwr` and `csrxchg`
(LoongArch Vol. 1 §4.2.1) are in the registry for bare-metal code.
`Builder.CSRRD`, `CSRWR` and `CSRXCHG` emit them, and tamago's bare-metal
loong64 port writes the same `WORD`s by hand, for example
`WORD $0x04000833 // CSRWR $2, R19` to enable the FPU. The registry has 19
reference cases from GNU as 2.44, and llvm-mc 23 agrees on all of them. The
three instructions share one opcode, and the rj field tells them apart: 0 is
`csrrd`, 1 is `csrwr`, anything else is `csrxchg`. So an entry can also exclude
encodings from its mask (`NotMask`/`NotBits`), and `scan` does not mistake
one for another.

The loong64 float32 fused multiply-add, `vfmadd.s` and `xvfmadd.s`, is in the
registry since v0.23.0. `Builder.VFMADDF` and `Builder.XVFMADDF` emit it as a
`WORD`, with the spelling CL 845165 gives it. The encodings come from
llvm-mc 23.1.2, which also reproduces the held GNU as words for `vfmadd.d` and
`xvfmadd.d`, and equal the cases the mailed patch holds. go-ndarray's float32
GEMM on loong64 uses them.

### arm64 multiply by element {#arm64-multiply-by-element}

`cmd/asm` has the vector-by-vector `VFMLA`/`VFMLS`/`VFMUL` but not their
indexed forms, `fmla vd.2d, vn.2d, vm.d[i]`. In those, every lane is multiplied
by one lane of Vm, the broadcast a GEMM micro-kernel needs. go-ndarray's GEMM
kernel therefore writes sixteen of them as `WORD`s by hand. The registry holds
fmla, fmls and fmul in `.4S` and `.2D`, spelt the way
[CL 764224](https://go.dev/cl/764224) (Jonathan Swinney) spells them:

```
WORD $0x4fd41200 // VFMLA V20.D[0], V16.D2, V0.D2
```

The 76 reference cases come from llvm-mc 23, and GNU as 2.44 agrees on every
one. They include go-ndarray's sixteen words. Vm is any of V0–V31 for `.S` and
`.D`; only the half-precision forms are limited to V0–V15. The index is 0–3
for `.S` and 0–1 for `.D`.

| Toolchain | `goasmgap verify -arch arm64` |
|---|---|
| Go 1.27.1 | 6/6 `ABSENT` |
| master + CL 764224 patchset 6 | 6/6 `SUPPORTED`, 76/76 identical |

Patchset 6 is stacked on two other CLs by the same author and does not apply to
master by itself, but the three apply cleanly together. It is not one of our
CLs, so it is not in `goasm-patches/cls.txt`: the weekly job would apply it
without its parents and would report its author's review threads as ours. The
weekly release and master checks still report the day these forms assemble.

## goasmgap {#goasmgap}

`tools/goasmgap` works from the registry:

```sh
go run ./tools/goasmgap scan -goroot ~/go-master       # Go's test data, searched by encoding
go run ./tools/goasmgap testdata -arch ppc64           # cmd/asm test lines for a patch
go run ./tools/goasmgap verify -goroot ~/go-master -require
```

- **`scan` searches by encoding, not by name.** It looks for the opcode bits
  under whatever name Go uses. This is how the loong64 broadcast load turned out
  to be in `cmd/asm` all along, as an arrangement load, after two searches by
  name had concluded it was missing.
- **`verify` assembles every reference case** and reports `SUPPORTED`,
  `PARTIAL`, `ABSENT` or `WRONG`. It builds `cmd/asm` from the tree's source, so
  an out-of-date installed binary cannot pass for a patch. Before it trusts a
  refusal, it checks that the toolchain can assemble at all.

## Patches and the weekly check {#patches-and-the-weekly-check}

[`goasm-patches/`](https://github.com/go-asmgen/asmgen/tree/main/goasm-patches)
holds the patches for Go. Each one is checked against independent assemblers
and passes Go's own tests. The first three were mailed on 2026-10-05:

- [CL 845145](https://go.dev/cl/845145): ppc64 VSX float64 arithmetic.
  **Merged on 2026-10-20**; it ships in Go 1.28. go-asmgen keeps emitting
  `WORD`s for these instructions until its consumers build with Go 1.28, since
  its floor is Go 1.27;
- [CL 845165](https://go.dev/cl/845165): loong64 vector FMA (with its 14
  siblings) and the full offset range of the broadcast loads;
- [CL 845166](https://go.dev/cl/845166): a range check for loong64 element
  stores. On Go master these silently encoded an out-of-range offset as a
  store to the wrong address.
- [CL 846285](https://go.dev/cl/846285): ppc64 VSX float32 arithmetic, the
  counterpart of CL 845145 (mailed 2026-10-07). **Merged on 2026-10-29**; it
  ships in Go 1.28, and the same applies: `WORD`s until consumers build with
  Go 1.28;
- [CL 846286](https://go.dev/cl/846286): s390x vector float32 (`VFASB`…,
  vector-enhancements facility 1), which go-simd/floats needs to vectorise
  float32 on s390x (mailed 2026-10-07);
- [CL 853786](https://go.dev/cl/853786): loong64 CSR access, `CSRRD $csr, Rd`,
  `CSRWR $csr, Rd` and `CSRXCHG $csr, Rj, Rd`. It refuses a CSR number above
  16383, and a `CSRXCHG` mask register of R0 or R1, which would encode the
  other two (mailed 2026-10-25).

Every Monday, the `goasm-gaps` workflow:

- verifies the latest Go release and Go master, and opens an issue when either
  fully assembles an instruction in the registry;
- reads each mailed CL on Gerrit, and opens an issue when one has review
  comments waiting for a reply or was abandoned. A mailed CL is forgotten the
  same way as an unmailed patch: a reviewer asks something and nobody answers;
- applies every patch to master, taking a mailed CL as its current patchset,
  and requires Go's tests and every reference encoding to pass. A merged or
  abandoned CL is not applied: master already holds a merged one.
