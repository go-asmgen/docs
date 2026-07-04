# wasm — Design notes

## Same layering, different bottom

The [main asmgen module](../asmgen/design.md) splits Plan 9 assembly
generation into three layers:

- `abi` — architecture-independent ABI0 frame layout.
- Per-arch builders (`amd64` / `arm64` / …) — thin move/register tables over
  the shared layout.
- `emit` — writes `.s` file text: the `TEXT` block, `#include`s, `//go:build`
  header, instruction lines.

The `wasm` package inside `go-asmgen/asmgen` slots into that architecture
as a peer of the six per-arch builders — a **seventh target** — but with
two structural differences forced by wasm itself:

1. **Not a shared-ABI target.** The wasm ABI is stack-based (locals plus a
   value stack), not the FP/SP/register-file model ABI0 codifies. The
   `abi.Signature` layout logic that all six Plan 9 arches share does not
   apply.
2. **Not a `cmd/asm` target.** The Go toolchain assembler has no wasm
   dialect; a WAT-to-wasm translation step (`wat2wasm`) sits between the
   emitter and the wasm binary the consumer imports.

So the module is small: one package, one `emit.go` file, no shared-abi
plumbing. The emitter builds a `Module` (the `(module ...)` s-expression),
each `Function` is a `(func $name (export "name") ...)` body, and the ops
push/pop against wasm's implicit value stack in evaluation order.

## Why a text emitter, not a wasm-binary encoder

Same argument as the main module. Encoding wasm bytes directly would require
tracking type IDs, section offsets, function indices, LEB128 varints — a full
wasm binary encoder, hundreds of lines of infrastructure. Emitting WAT text
and delegating to `wat2wasm` is:

- **Smaller.** The whole emit surface is ~300 lines including comments.
- **Debuggable.** A human can read the generated `.wat` and diff it against
  hand-authored source.
- **Stable.** WABT tracks the wasm spec upstream; the WAT surface stays in
  sync with wasm features (SIMD, GC, exceptions, …) without emitter changes.
- **Composable with tooling.** `wasm2wat`, `wasm-validate`, wasm-opt all
  interoperate with the emitter output as-is.

## Stack semantics as an emit model

Every op the emitter exposes is a wasm stack op — its operands must already
be on the value stack, and its result gets pushed. The kernel author writes
them in evaluation order, and each Go call maps to one line of WAT:

```go
// eq = i8x16.eq( v128.load(ptrA + i), v128.load(ptrB + i) )
fn.LocalGet("ptrA")
fn.LocalGet("i")
fn.I32Add()
fn.V128Load(0)
fn.LocalGet("ptrB")
fn.LocalGet("i")
fn.I32Add()
fn.V128Load(0)
fn.I8x16Eq()
fn.LocalSet("eq")
```

That reads exactly like the hand-authored WAT it produces, one line per
s-expression. The `Function.String()` renderer indents everything one level
inside the `(func ...)` block; blocks and loops open/close via `Block` /
`Loop` / `End`.

## What the emitter does NOT model

- **Block signatures.** A `(block (result i32) ...)` variant would let a
  block carry a stack result across its boundary. All five shipped kernels
  route control flow through `(block ...)` and `(loop ...)` blocks with
  empty signatures; when a value is needed on exit, they either compute it
  outside the block or `(return)` directly. Adding block signatures is a
  drop-in extension.
- **`if`/`else`.** The shipped kernels model conditionals as
  `block { br_if end; ... }` — a "skip if" pattern. Native `if`/`else` would
  produce more idiomatic WAT for some kernels; it's not yet wired.
- **Multi-value returns.** The wasm `multi-value` proposal is baseline in
  every runtime, but no shipped kernel needs it, so it is not modeled.
- **wasm-GC, exception handling, threads, memory64, custom page sizes.**
  All post-MVP proposals; none in scope for the current kernels.

Every one of these is a "when needed" addition, not a design tension.

## Verifying an emitted kernel

Each kernel package ships three verification layers:

- **Unit tests on the emitter surface** (`emit_test.go`) — 100% coverage of
  the ops family and their formatting.
- **Golden-file test per kernel** — the current `.wat.golden` is pinned via
  `go test`, and `-update` rewrites it. Any generator change that shifts a
  byte of WAT trips this immediately.
- **wazero cross-check** (`verify/`) — the compiled `.wasm` runs under
  wazero's interpreter/JIT and its outputs are compared against a Go
  reference (`bytes.Equal`, `encoding/hex`, `math/bits`, `strings.ToUpper`,
  `bytes.IndexByte`).

The CI wires all three together: `.github/workflows/ci.yml` runs the emitter
suite on native and under qemu across all 6 Go 64-bit arches; the
`wasm-e2e` job installs WABT, regenerates each kernel, drift-gates against
the golden, compiles, and runs the wazero verifier.

## The consumer side

Consumers (see [`go-simd/matchlen-wasm`](https://github.com/go-simd/matchlen-wasm))
pin the generator version in a package-scoped `//go:generate` directive, run
it as part of `go generate ./...`, then compile with `wat2wasm`. Their own
CI adds two more gates:

- **wasm-drift** — regenerates and diffs the committed `.wat` / `.wasm`
  against the pinned generator output. Silent divergence is impossible.
- **wasm-bench** — runs the kernel under wazero, uploads a bench baseline on
  main, and PRs get a `benchstat` diff plus a fail gate on statistically
  significant regression.

Both are ~200-line workflows and the shape is stable across kernels — a
consumer for `hex` or `popcount` would copy them wholesale.
