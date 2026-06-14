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
sig := ppc64le.Layout(
    []string{"a", "b"}, []ppc64le.Type{ppc64le.Int64, ppc64le.Int64},
    []string{"ret"}, []ppc64le.Type{ppc64le.Int64},
)
b := ppc64le.NewFunc("add", sig, 0)
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
qemu-user, in a dedicated `asm-ppc64le` CI job on `debian:trixie`:

```bash
go generate ./examples/ppc64le/...
GOOS=linux GOARCH=ppc64le go vet ./examples/ppc64le/...
GOOS=linux GOARCH=ppc64le go build ./examples/ppc64le/...
QEMU_CPU=power9 GOARCH=ppc64le go test -exec=qemu-ppc64le-static ./examples/ppc64le/...
```
