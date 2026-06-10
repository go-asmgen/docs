# asmgen — ABI0 & design notes

## Why ABI0, not ABIInternal

Go has two calling conventions:

- **ABIInternal** — the register-based convention the compiler uses internally.
  It is an *unstable* contract that can change between Go releases.
- **ABI0** — the stack-based, FP-relative convention. Arguments and results live
  on the stack, addressed via the `FP` pseudo-register in declaration order.
  This is the stable contract that hand-written `.s` files target.

go-asmgen targets **ABI0**. That is what makes the emitted assembly safe to
check in and rely on across Go versions, and it is what `go vet`'s asmdecl pass
understands.

## The ABI0 layout model

`arm64.Layout` lays out arguments, then results, contiguously. Each value is
placed at an offset that respects its own alignment, and the total
argument+result area is rounded up to the register word size (8 bytes on arm64):

```go
func align(n, to int) int { return (n + to - 1) &^ (to - 1) }
```

For `func add(a, b int64) int64` that yields:

| Slot | Size | Offset |
| --- | --- | --- |
| `a` | 8 | 0 |
| `b` | 8 | 8 |
| `ret` | 8 | 16 |

so the `TEXT` directive is `$0-24` (frame 0, args+results 24) and the loads/stores
use `a+0(FP)`, `b+8(FP)`, `ret+16(FP)`.

## What v0 is — and is not — correct for

!!! warning "Scope of v0"
    v0 is a **proof of pipeline**. It is correct only for sequences of 8-byte
    (`int64` / `uint64` / pointer) arguments and results under ABI0.

Not yet correct for:

- mixed-size arguments (1/2/4-byte ints) and the padding rules they need,
- struct or array arguments,
- floating-point or vector values (which use the `F`/`V` register files and
  `FMOV*` moves),

The layout function already aligns per declared size, but `LoadArg`/`StoreRet`
currently hardcode `MOVD` (the 8-byte move), so sub-word and float values need
move-instruction selection before they are correct. Each new case must be
checked against `go vet` asmdecl, which cross-checks `.s` offsets against the Go
declaration. See the [Roadmap](roadmap.md).

## NOSPLIT by default

v0 marks every function `NOSPLIT`, which omits the stack-growth preamble. That
is fine for small leaf functions with no locals, and keeps the emitted assembly
minimal. It must be revisited for functions with large frames or that call other
functions.

## Validation priorities

The asmgen CI encodes the correctness order:

1. **`go vet` asmdecl** — offsets in the `.s` match the Go declaration.
2. **Generated assembly is committed** — CI regenerates and fails on any diff.
3. **Runtime test on native arm64** — the function is actually called and its
   result checked.
