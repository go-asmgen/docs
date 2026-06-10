# Contributing

go-asmgen follows the same conventions as the rest of the author's Go
organizations.

## Hard rules

- **Pure Go, `CGO_ENABLED=0`** for the library — it is ordinary,
  architecture-independent Go that builds everywhere.
- **100% test coverage** of the library packages (`abi`, `emit`, and the four
  builders `amd64`/`arm64`/`riscv64`/`loong64`), enforced as a CI gate. Every
  branch must be reachable from a test, or the dead code is removed — the bar is
  never lowered.
- **Correctness is proven, not asserted.** A new instruction or type case is not
  "done" until `go vet` asmdecl passes on the generated `.s` *and* a runtime
  test exercises it — natively on amd64 and arm64, under qemu-user for riscv64
  and loong64.
- **English only** for all repository content (issues, PRs, commits, code
  comments).

## Adding a type or instruction

1. Extend the layout (`abi`) and/or the architecture's move table.
2. Add an example that generates the new assembly, with a runtime test.
3. Validate the example on its target — for arm64 (natively):
   ```bash
   go generate ./examples/...
   GOARCH=arm64 go vet ./examples/add/...    # asmdecl: offsets vs Go declaration
   go test ./examples/add/...                # runtime
   ```
   For riscv64/loong64, cross-vet + `cmd/asm` build, then run under qemu-user.
4. Confirm the library still measures 100% coverage:
   ```bash
   go test -coverprofile=cover.out ./abi/... ./emit/... ./amd64/... ./arm64/... ./riscv64/... ./loong64/...
   go tool cover -func=cover.out
   ```

## Adding an architecture

A new 64-bit ISA package reuses both `abi` (the shared ABI0 layout) and `emit`
unchanged, and provides only:

- a **move table** — `loadMnemonic`/`storeMnemonic` for the ISA's registers, and
- a thin `Builder` (`LoadArg`/`StoreRet`/`Raw`/`Ret`) that re-exports the `abi`
  types.

Arithmetic and vector ops go through `Raw`; encoding stays delegated to
`cmd/asm`. riscv64 and loong64 were each exactly this — see their pages under
the asmgen section.

## Documentation

This site is built with MkDocs Material and versioned with
[mike](https://github.com/jimporter/mike); see the
[docs repo README](https://github.com/go-asmgen/docs) for local preview and
release commands.
