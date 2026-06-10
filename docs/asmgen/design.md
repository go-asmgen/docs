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

## What's supported

All four targets handle sequences of **scalars** in any combination — signed and
unsigned integers of 1/2/4/8 bytes, pointers, and 32/64-bit floats — plus:

- **struct, slice, and string** parameters — see [Aggregates](aggregates.md);
- **fixed-size arrays** passed by value, addressed element-wise — see
  [Aggregates › Fixed-size arrays](aggregates.md#fixed-size-arrays);
- **SIMD** (SSE/NEON/RVV/LSX) emitted through `Raw` over pointer arguments — see
  [SIMD](simd.md).

The same model drives [amd64](amd64.md), [arm64](index.md), [riscv64](riscv64.md)
and [loong64](loong64.md).

!!! warning "Not yet"
    First-class **vector types** in the typed surface — the builders' `LoadArg`/
    `StoreRet` stop at scalars, so SIMD is written via `Raw`. A single vector
    load of a whole by-value array is not asmdecl-clean (see [SIMD](simd.md)).

Each new case is checked against `go vet` asmdecl, which cross-checks `.s`
offsets and access widths against the Go declaration. See the
[Roadmap](roadmap.md).

## TEXT flags and stack frames

`NewFunc` marks every function `NOSPLIT`, which omits the stack-growth preamble —
fine for small leaf functions and the common case. For anything else, **`NewFuncFlags`**
takes explicit Plan 9 TEXT flags:

```go
b := arm64.NewFuncFlags("f", sig, 16, "")            // 16-byte frame, no NOSPLIT
b := arm64.NewFuncFlags("g", sig, 0, "NOSPLIT|NOFRAME")
```

Passing `""` omits the flags so the assembler inserts the stack-growth preamble
(needed for large frames or non-leaf functions). `frameSize > 0` reserves stack
locals, addressed `name-N(SP)` via `Raw`. See
[`examples/frame`](https://github.com/go-asmgen/asmgen/tree/main/examples/frame),
which spills its arguments to a 16-byte frame, non-`NOSPLIT`, runtime-proven.

## Validation priorities

The asmgen CI encodes the correctness order:

1. **`go vet` asmdecl** — offsets in the `.s` match the Go declaration.
2. **Generated assembly is committed** — CI regenerates and fails on any diff.
3. **Runtime test** — the function is actually called and its result checked:
   natively on amd64 and arm64, and under qemu-user for riscv64 and loong64.
