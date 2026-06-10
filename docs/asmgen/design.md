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

### The result area is word-aligned

One subtlety asmdecl enforces: the **result area starts at an offset rounded up
to the register word size** (8 on arm64), and the total frame size is the end of
the last result — it is *not* rounded further. So `func add16(a, b int16) int16`
lays out `a@0`, `b@2`, but `ret@8` (not `ret@4`), with `TEXT $0-10`:

| Slot | Size | Offset |
| --- | --- | --- |
| `a` | 2 | 0 |
| `b` | 2 | 2 |
| `ret` | 2 | 8 |

Getting this wrong is exactly the kind of mistake `go vet` asmdecl catches
before runtime.

## Move selection

`LoadArg`/`StoreRet` pick the Plan 9 move from the parameter's `Type`:

| Type | load | store |
| --- | --- | --- |
| `int8` / `int16` / `int32` | `MOVB` / `MOVH` / `MOVW` (sign-extend) | `MOVB` / `MOVH` / `MOVW` |
| `uint8` / `uint16` / `uint32` | `MOVBU` / `MOVHU` / `MOVWU` (zero-extend) | `MOVB` / `MOVH` / `MOVW` |
| `int64` / `uint64` / pointer | `MOVD` | `MOVD` |
| `float32` / `float64` | `FMOVS` / `FMOVD` | `FMOVS` / `FMOVD` |

Sub-word loads use the sign- or zero-extending form so the whole register holds
the correct value; floats use the `F` register file. Arithmetic is still supplied
via the `Raw` escape hatch (e.g. `ADDW`, `FADDD`).

## What v0 is — and is not — correct for

v0 is correct for sequences of arm64 **scalars** in any combination — signed and
unsigned integers of 1/2/4/8 bytes, pointers, and 32/64-bit floats.

Struct, slice, and string parameters are also supported — see
[Aggregates](aggregates.md).

!!! warning "Not yet correct for"
    - **array** arguments/results, and
    - **vector** values (the `V` register file).

Each new case must be checked against `go vet` asmdecl, which cross-checks `.s`
offsets and access widths against the Go declaration. See the
[Roadmap](roadmap.md).

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
