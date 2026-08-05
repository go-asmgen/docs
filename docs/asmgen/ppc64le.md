# Fifth target: ppc64le

ppc64le (POWER, little-endian) is the fifth architecture. Like the others, it
reuses the shared ABI0 model (`abi`) unchanged and adds only a move table.

## The move table

| Type | arm64 | ppc64le |
| --- | --- | --- |
| `int64` / `uint64` / pointer | `MOVD` | `MOVD` |
| `int8`/`int16`/`int32` (signed) | `MOVB`/`MOVH`/`MOVW` | `MOVB`/`MOVH`/`MOVW` |
| `uint8`/`uint16`/`uint32` | `MOVBU`/`MOVHU`/`MOVWU` | `MOVBZ`/`MOVHZ`/`MOVWZ` |
| `float32` | `FMOVS` | `FMOVS` |
| `float64` | `FMOVD` | `FMOVD` |

ppc64le's 8-byte integer move is `MOVD`, and its floats use `FMOVS`/`FMOVD` — the
same shape as arm64. The unsigned sub-word loads use the `Z` (zero-extend)
suffix (`MOVBZ`/`MOVHZ`/`MOVWZ`). The ABI0 offsets are byte-for-byte identical to
the other targets.

## Example

```go
// package ppc64 targets ppc64le
sig := ppc64.Layout(
    []string{"a", "b"}, []ppc64.Type{ppc64.Int64, ppc64.Int64},
    []string{"ret"}, []ppc64.Type{ppc64.Int64},
)
b := ppc64.NewFunc("add", sig, 0)
b.LoadArg("a", "R3").
    LoadArg("b", "R4").
    Raw("ADD R4, R3, R5").
    StoreRet("R5", "ret").
    Ret()
```

emits:

```asm
TEXT ·add(SB), NOSPLIT, $0-24
	MOVD a+0(FP), R3
	MOVD b+8(FP), R4
	ADD R4, R3, R5
	MOVD R5, ret+16(FP)
	RET
```

## SIMD

The packed-add example uses VSX (`LXVD2X` / `VADDUWM` / `STXVD2X`) through `Raw`
over pointer arguments — see [SIMD](simd.md).

## Validation

There is no common ppc64le developer host, so correctness is established in
layers — asmdecl (cross-arch), `cmd/asm` cross-build, and runtime under
qemu-user (`qemu-user-static`), in a dedicated `asm-ppc64le` CI job on
`ubuntu-latest`. The default qemu CPU already models VSX (baseline on POWER8+),
so no `QEMU_CPU` override is needed. The package is `ppc64` (import path
`github.com/go-asmgen/asmgen/ppc64`); only its own SIMD example is
runtime-proven today (see [SIMD](simd.md) — no dedicated aggregate/array
example yet, see the [Roadmap](roadmap.md)):

```bash
go generate ./examples/simd/ppc64/...
GOOS=linux GOARCH=ppc64le go vet ./examples/simd/ppc64/...
GOOS=linux GOARCH=ppc64le go build ./examples/simd/ppc64/...
GOARCH=ppc64le go test -exec=qemu-ppc64le-static ./examples/simd/ppc64/...
```
