# Security

go-asmgen runs at `go generate` time, on input written by the generator's
author. It has no network-facing surface. Its risk lies in what it **emits**:
the committed assembly runs in every module that uses it, outside Go's memory
safety.

## What counts as a vulnerability

- **An encoding that is not the instruction it names.** A `WORD` (see
  [Instructions cmd/asm lacks](gaps.md)) is the one place `cmd/asm` checks
  nothing. Each encoder is pinned to reference encodings from GNU as, and
  refuses any register or offset that does not encode. An operand it accepts
  but encodes wrongly is a vulnerability.
- **A frame or argument layout that disagrees with the Go declaration** in a
  way `go vet` (asmdecl) does not catch.
- **A builder that emits other than it documents:** another register, width or
  offset, or a clobbered callee-saved register.

Text passed to `Raw` is emitted as written. Checking it is the caller's job, as
for hand-written assembly: run `go vet` and build for the target.

## Reporting

Report privately, through the repository's
[Report a vulnerability](https://github.com/go-asmgen/asmgen/security/advisories/new)
form, not in a public issue. The policy is in
[SECURITY.md](https://github.com/go-asmgen/asmgen/blob/main/SECURITY.md).

## Audit of 2026-10-04 (v0.15.0)

| Check | Result |
|---|---|
| govulncheck | no vulnerability; the module has no dependencies. CI runs it on every change. |
| gosec | conversions in the encoders are each preceded by a range check; no change needed |
| staticcheck | one deprecated call (`runtime.GOROOT`) in a test |
| Encoder rejections | register 32, offset not a multiple of 8, offset out of range: all tested |
| CI workflows | read-only token (`issues: write` only in the scheduled gap report, never on a pull request); no `pull_request_target`; no secrets |
| Repositories | private vulnerability reporting, secret scanning and push protection enabled on all five |
