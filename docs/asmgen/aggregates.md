# Aggregates — structs, slices, strings

ABI0 passes structs, slices, and strings **by value on the stack**, laid out as
their constituent fields. go-asmgen models this in the shared layout: an
aggregate parameter is flattened into one slot per field, named
`param_field`, which is exactly how hand-written Plan 9 assembly addresses it —
and exactly what `go vet` asmdecl checks.

## Field naming

| Go type | fields (slots) |
| --- | --- |
| `struct{A, B int64}` named `p` | `p_A`, `p_B` |
| `[]T` named `s` | `s_base`, `s_len`, `s_cap` |
| `string` named `x` | `x_base`, `x_len` |

A slice header is `{base *T; len, cap int}` (24 bytes); a string is
`{base *byte; len int}` (16 bytes). Struct fields follow Go's layout rules: each
at its natural alignment, the struct aligned to its widest field and sized with
trailing padding. So `struct{Flag int8; N int64}` places `N` at offset **8**, not
1 — the generated assembly must address the padded offset, and asmdecl fails the
build if it doesn't.

## Describing aggregates

Layout takes aggregate arguments via `abi.LayoutArgs` and the `Struct` / `Slice`
/ `String` constructors:

```go
sig := abi.LayoutArgs(
    []abi.Arg{abi.Slice("s")},          // s_base@0, s_len@8, s_cap@16
    []abi.Arg{abi.Scalar("ret", abi.Int64)},
)
```

The arch builders need no aggregate-specific API: a field is just a named slot,
so `LoadArg("s_base", reg)` loads the base pointer with the right move, and
dereferencing it (`s[0]`) is ordinary arithmetic through the `Raw` hatch.

## Worked example (arm64)

`func sliceFirst(s []int64) int64 { return s[0] }`:

```go
sig := abi.LayoutArgs([]abi.Arg{abi.Slice("s")}, []abi.Arg{abi.Scalar("ret", abi.Int64)})
b := arm64.NewFunc("sliceFirst", sig, 0)
b.LoadArg("s_base", "R0").   // base pointer from the slice header
    Raw("MOVD (R0), R1").    // dereference: R1 = s[0]
    StoreRet("R1", "ret").
    Ret()
```

emits:

```asm
TEXT ·sliceFirst(SB), NOSPLIT, $0-32
	MOVD s_base+0(FP), R0
	MOVD (R0), R1
	MOVD R1, ret+24(FP)
	RET
```

The result sits at offset 24 — after the 24-byte slice header, word-aligned. The
[`examples/aggregate`](https://github.com/go-asmgen/asmgen/tree/main/examples/aggregate)
package also covers a by-value struct, a padded struct, a slice length, and a
string length, each runtime-tested on native arm64.

## Why asmdecl is the safety net

asmdecl independently computes the Go layout of every parameter — including the
internal structure of slices, strings, and structs — and rejects any
`name_field+offset(FP)` whose offset or width does not match. That makes the
padded-field and slice-header offsets self-checking: if go-asmgen computed them
wrong, `go vet` would fail before any code ran.

## Fixed-size arrays

A `[n]T` array passed **by value** is laid out the same way — `abi.Array` flattens
it into element slots `name_0 … name_(n-1)`, exactly the names asmdecl gives array
elements:

```go
sig := abi.LayoutArgs(
    []abi.Arg{abi.Array("v", abi.Int64, 3)},   // v_0@0, v_1@8, v_2@16
    []abi.Arg{abi.Scalar("ret", abi.Int64)},
)
b.LoadArg("v_0", "R0").LoadArg("v_1", "R1").LoadArg("v_2", "R2"). /* … */
```

See [`examples/array`](https://github.com/go-asmgen/asmgen/tree/main/examples/array).
`abi.Signature.Slot(name)` looks up any slot's offset, handy when you need an
aggregate's base offset for a `Raw` access.

!!! note "By-value arrays are accessed element-wise"
    `go vet` asmdecl does **not** accept a single wide/vector load of a whole
    by-value array — e.g. `MOVOU a+0(FP), X0` for a `[4]int32` is rejected
    ("invalid MOVOU … is 16-byte value"). Idiomatic Go SIMD therefore passes
    vector data **by pointer**; see [SIMD](simd.md).

## Not yet covered

First-class vector *types* in the typed surface (SIMD is via `Raw`). See the
[Roadmap](roadmap.md).
