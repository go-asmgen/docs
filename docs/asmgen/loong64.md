# Third architecture: loong64

loong64 (LoongArch 64-bit) was the cheapest architecture yet — it confirms that
the [riscv64](riscv64.md) drop-in was not a coincidence between two similar
ISAs. It reuses the shared ABI0 model (`abi`) unchanged and adds only a
move table.

## The move table

| Type | arm64 | riscv64 | loong64 |
| --- | --- | --- | --- |
| `int64` / `uint64` / pointer | `MOVD` | `MOV` | **`MOVV`** |
| `int8`/`int16`/`int32` (signed) | `MOVB`/`MOVH`/`MOVW` | same | same |
| `uint8`/`uint16`/`uint32` | `MOVBU`/`MOVHU`/`MOVWU` | same | same |
| `float32` | `FMOVS` | `MOVF` | `MOVF` |
| `float64` | `FMOVD` | `MOVD` | `MOVD` |

loong64's 8-byte integer move is `MOVV` (the "V" = 8-byte/doubleword, inherited
from the MIPS lineage of Go's assembler). Floats match riscv64 (`MOVF`/`MOVD`).
The ABI0 offsets are, again, byte-for-byte identical to the other targets.

## Example

```go
sig := loong64.Layout(
    []string{"a", "b"}, []loong64.Type{loong64.Int64, loong64.Int64},
    []string{"ret"}, []loong64.Type{loong64.Int64},
)
b := loong64.NewFunc("add", sig, 0)
b.LoadArg("a", "R4").
    LoadArg("b", "R5").
    Raw("ADDV R5, R4, R6").
    StoreRet("R6", "ret").
    Ret()
```

emits:

```asm
TEXT ·add(SB), NOSPLIT, $0-24
	MOVV a+0(FP), R4
	MOVV b+8(FP), R5
	ADDV R5, R4, R6
	MOVV R6, ret+16(FP)
	RET
```

## A note on arithmetic and `cmd/asm`

Arithmetic is supplied through the `Raw` escape hatch, and `cmd/asm` is the
backstop that keeps it honest. loong64's assembler has no three-register `ADDW`
form, so the int32 example uses a 64-bit `ADDV` with a `MOVW` store (the low 32
bits of the sum are the correct `int32` result). The build step caught the wrong
mnemonic immediately — the move-selection surface that go-asmgen *does* model
(`MOVW` load/store) is unaffected.

## Validation

Identical to riscv64 — asmdecl (cross-arch), `cmd/asm` cross-build, and runtime
under qemu-user, in a dedicated `asm-loong64` CI job:

```bash
go generate ./examples/loong64/...
GOOS=linux GOARCH=loong64 go vet ./examples/loong64/...
GOOS=linux GOARCH=loong64 go build ./examples/loong64/...
GOARCH=loong64 go test -exec=qemu-loongarch64-static ./examples/loong64/...
```
