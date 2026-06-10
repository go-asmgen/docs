# SIMD (SSE / NEON / …)

go-asmgen does not model vector instructions in its typed surface, and it does
not need to: vector code is emitted through the **`Raw` escape hatch**, exactly
as scalar arithmetic is. go-asmgen's job is to lay out the ABI0 frame and load
the arguments; the packed body is yours to write.

The idiomatic shape is a function that takes **pointers** to the data, so the
ABI0 frame is just pointer slots and the vector loads/stores go through those
registers:

```go
// func addI32x4(a, b, out *[4]int32)
sig := arm64.Layout(
    []string{"a", "b", "out"},
    []arm64.Type{arm64.Ptr, arm64.Ptr, arm64.Ptr},
    nil, nil,
)
b := arm64.NewFunc("addI32x4", sig, 0)
b.LoadArg("a", "R0").
    LoadArg("b", "R1").
    LoadArg("out", "R2").
    Raw("VLD1 (R0), [V0.S4]").       // NEON: load 4 int32 lanes
    Raw("VLD1 (R1), [V1.S4]").
    Raw("VADD V0.S4, V1.S4, V2.S4"). // packed add
    Raw("VST1 [V2.S4], (R2)").       // store 4 lanes
    Ret()
```

The amd64 equivalent is the same structure with SSE2:

```asm
MOVQ   a+0(FP), AX
MOVQ   b+8(FP), BX
MOVQ   out+16(FP), CX
MOVOU  (AX), X0      // load 4 int32 lanes
MOVOU  (BX), X1
PADDL  X1, X0        // packed add, dword lanes
MOVOU  X0, (CX)      // store
RET
```

Both are in [`examples/simd`](https://github.com/go-asmgen/asmgen/tree/main/examples/simd),
runtime-tested: amd64 natively (the GitHub runner is amd64), arm64 natively on
Apple Silicon / arm64 Linux.

## What asmdecl checks here

Only the **pointer loads** are FP-relative, so asmdecl validates those (three
8-byte pointer reads at offsets 0/8/16). The vector loads/stores go through
registers (`(AX)`, `(R0)`), which asmdecl does not track — their correctness is
established by the **runtime test**, which checks every lane.

## SIMD support across the targets

Every 64-bit Go target the project covers has vector support in the assembler,
so the same `Raw`-based approach extends to all of them:

| Target | SIMD in `cmd/asm` |
| --- | --- |
| amd64 | SSE/SSE2 (`PADDL`, `ADDPS`, `MOVOU`…) and AVX (`VADDPS`, Y/Z registers) |
| arm64 | NEON/ASIMD (`VADD`, `VLD1`, V registers with `.S4`/`.D2`… arrangements) |
| riscv64 | RVV — the RISC-V Vector extension (`VSETVLI`, `VLE32.V`, V registers) |
| loong64 | LSX (128-bit, `V*`) and LASX (256-bit, `XV*`) |

## Why pointers, not by-value vectors

`go vet` asmdecl rejects a single vector load of a whole by-value array — e.g.
`MOVOU a+0(FP), X0` for a `[4]int32` argument fails with *"invalid MOVOU …;
[4]int32 is 16-byte value"*. So passing a vector **by value** and loading it in
one instruction is not asmdecl-clean. The pointer-based shape above is the
idiomatic Go approach and is what go-asmgen supports. (Fixed-size arrays passed
by value are still usable **element-wise** — see [Aggregates](aggregates.md#fixed-size-arrays).)

## Future work

A typed vector-load helper could emit the pointer load/op/store sequence for you,
removing the `Raw` boilerplate. See the [Roadmap](roadmap.md); the `Raw` path
works today.
