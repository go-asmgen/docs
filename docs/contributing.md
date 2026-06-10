# Contributing

go-asmgen follows the same conventions as the rest of the author's Go
organizations.

## Hard rules

- **Pure Go, `CGO_ENABLED=0`** for the library — it is ordinary,
  architecture-independent Go that builds everywhere.
- **100% test coverage** of the library packages (`arm64`, `internal/emit`),
  enforced as a CI gate. Every branch must be reachable from a test, or the dead
  code is removed — the bar is never lowered.
- **Correctness is proven, not asserted.** A new instruction or type case is not
  "done" until `go vet` asmdecl passes on the generated `.s` *and* a runtime
  test exercises it on native arm64.
- **English only** for all repository content (issues, PRs, commits, code
  comments).

## Adding a type or instruction

1. Extend the layout/emit logic.
2. Add an example or test that generates the new assembly.
3. Run, in order:
   ```bash
   go generate ./...
   GOARCH=arm64 go vet ./...   # asmdecl: offsets vs Go declaration
   GOARCH=arm64 go build ./...
   go test ./...               # native arm64
   ```
4. Confirm the library still measures 100% coverage:
   ```bash
   go test -coverprofile=cover.out ./arm64/... ./internal/...
   go tool cover -func=cover.out
   ```

## Adding an architecture

A new ISA package reuses `internal/emit` unchanged and provides:

- an **ABI0 layout model** (offsets, alignment, word size), and
- a **thin instruction-emit surface** (the moves and arithmetic you need).

Encoding stays delegated to `cmd/asm`. See the
[ABI0 & design notes](asmgen/design.md) for the contract a new architecture must
honor.

## Documentation

This site is built with MkDocs Material and versioned with
[mike](https://github.com/jimporter/mike); see the
[docs repo README](https://github.com/go-asmgen/docs) for local preview and
release commands.
