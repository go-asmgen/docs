# Instructions cmd/asm lacks

go-asmgen emits Plan 9 assembly and lets the Go assembler, `cmd/asm`, encode it.
Sometimes a kernel needs an instruction `cmd/asm` cannot assemble yet. go-asmgen
then emits it as a `WORD` for the time being, and keeps everything needed to
remove that `WORD` later.

## What gets emitted

```
WORD $0xf001130f // XVMADDADP VS33, VS34, VS32
```

The encoded word comes first. The comment is the spelling proposed for `cmd/asm`,
with operands in its order, so the line already shows what it will become.

As of v0.15.0:

| Architecture | Instructions | Emitted as |
|---|---|---|
| arm64 | `VFADD2D` `VFSUB2D` `VFMUL2D` `VFNEG2D` `VFMLA2D` `VFMLS2D` | mnemonics (Go 1.27) |
| loong64 | `VLDREPLD` `XVLDREPLD` | `VMOVQ off(R), V.V2` / `XVMOVQ off(R), X.V4`, except offset −2048 |
| loong64 | `VFMADDD` `XVFMADDD` | `WORD` |
| ppc64le | `XVADDDP` `XVSUBDP` `XVMULDP` `XVDIVDP` `XVMADDADP` `XVMAXDP` `XVMINDP` `XVSQRTDP` | `WORD` |

## The registry

Each `WORD` comes from an entry in `internal/gap`, which holds:

- the encoder;
- reference encodings from independent assemblers (GNU as on the real
  hardware; llvm-mc agrees on all 102 cases), covering every register field at
  its ends;
- the opcode mask;
- the proposed Plan 9 spelling.

The architecture packages only call into it, so there is one source.

## goasmgap

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

## Patches and the weekly check

[`goasm-patches/`](https://github.com/go-asmgen/asmgen/tree/main/goasm-patches)
holds the patches for Go. Each one is checked against independent assemblers
and passes Go's own tests. All three were mailed on 2026-10-05:

- [CL 845145](https://go.dev/cl/845145): ppc64 VSX float64 arithmetic;
- [CL 845165](https://go.dev/cl/845165): loong64 vector FMA (with its 14
  siblings) and the full offset range of the broadcast loads;
- [CL 845166](https://go.dev/cl/845166): a range check for loong64 element
  stores. On Go master these silently encoded an out-of-range offset as a
  store to the wrong address.

Every Monday, the `goasm-gaps` workflow:

- verifies the latest Go release and Go master, and opens an issue when either
  fully assembles an instruction in the registry;
- reads each mailed CL on Gerrit, and opens an issue when one has review
  comments waiting for a reply or was abandoned. A mailed CL is forgotten the
  same way as an unmailed patch: a reviewer asks something and nobody answers;
- applies every patch to master, taking a mailed CL as its current patchset,
  and requires Go's tests and every reference encoding to pass.
