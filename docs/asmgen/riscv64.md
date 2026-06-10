# Second architecture: riscv64

riscv64 is where go-asmgen's design pays off. Adding it required **no new layout
code** — it reuses the shared ABI0 model (`internal/abi`) unchanged, because
ABI0's stack layout is identical on every 64-bit target. All that differs is the
register file and the move table.

## What the `riscv64` package adds

Compared to `arm64`, the entire architecture-specific surface is the move
selection:

| Type | arm64 | riscv64 |
| --- | --- | --- |
| `int64` / `uint64` / pointer | `MOVD` | **`MOV`** |
| `int8`/`int16`/`int32` (signed) | `MOVB`/`MOVH`/`MOVW` | `MOVB`/`MOVH`/`MOVW` |
| `uint8`/`uint16`/`uint32` | `MOVBU`/`MOVHU`/`MOVWU` | `MOVBU`/`MOVHU`/`MOVWU` |
| `float32` | `FMOVS` | **`MOVF`** |
| `float64` | `FMOVD` | **`MOVD`** (the float double move) |

Note the deliberate trap: on riscv64 `MOVD` is the **float** double move, while
the 8-byte **integer** move is plain `MOV`. The opposite of arm64, where `MOVD`
is the integer move. Encoding each correctly is exactly what the per-arch move
table exists for — and `cmd/asm` rejects a wrong choice at build time.

The ABI0 offsets are byte-for-byte the same as arm64. For `func add(a, b int64)
int64` both emit `TEXT ·add(SB), NOSPLIT, $0-24` with `a+0(FP)`, `b+8(FP)`,
`ret+16(FP)` — only the mnemonic changes (`MOVD` → `MOV`).

## Example

```go
sig := riscv64.Layout(
    []string{"a", "b"}, []riscv64.Type{riscv64.Int64, riscv64.Int64},
    []string{"ret"}, []riscv64.Type{riscv64.Int64},
)
b := riscv64.NewFunc("add", sig, 0)
b.LoadArg("a", "X5").
    LoadArg("b", "X6").
    Raw("ADD X6, X5, X7").
    StoreRet("X7", "ret").
    Ret()
```

emits:

```asm
TEXT ·add(SB), NOSPLIT, $0-24
	MOV a+0(FP), X5
	MOV b+8(FP), X6
	ADD X6, X5, X7
	MOV X7, ret+16(FP)
	RET
```

## Validating riscv64

There is no common riscv64 developer host, so correctness is established in
layers:

1. **`go vet` asmdecl** (cross-arch) — verifies offsets and access widths
   against the Go declaration. Runs anywhere:
   ```bash
   GOOS=linux GOARCH=riscv64 go vet ./examples/riscv64/...
   ```
2. **`cmd/asm` cross-build** — rejects any invalid mnemonic or operand form:
   ```bash
   GOOS=linux GOARCH=riscv64 go build ./examples/riscv64/...
   ```
3. **Runtime under emulation** — the generated instructions are actually
   executed, in CI under qemu-user, or locally with Docker's riscv64 support:
   ```bash
   GOARCH=riscv64 go test -exec=qemu-riscv64-static ./examples/riscv64/...
   ```

The asmgen CI runs all three in a dedicated `asm-riscv64` job.

## Adding the next architecture

Any other 64-bit target follows the same recipe: re-export the shared `abi`
types, write a `loadMnemonic`/`storeMnemonic` pair, and reuse the `Builder`
shape. The layout model does not change — [loong64](loong64.md) was exactly
this.
