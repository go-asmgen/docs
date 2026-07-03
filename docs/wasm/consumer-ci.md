# wasm — Consumer CI pattern

A wasm consumer that pins one of the shipped kernels ends up owning three
CI concerns the emitter can't answer for them: **is the committed
`.wat` / `.wasm` still what the generator emits?** (drift), **has any PR
made the kernel slower under wazero?** (portable regression), and
optionally **has any PR made it slower on real hardware?** (arch-native
regression). This page documents the pattern that the reference
consumer, [`go-simd/matchlen-wasm`](https://github.com/go-simd/matchlen-wasm),
uses to answer all three, with the workflow shapes and the numbers
from a full end-to-end validation.

## The three-workflow layer

```text
                ┌──────────────────────────────────────────┐
   PR opens ──▶ │  wasm-drift        (always)              │
                │  regen .wat + wat2wasm → diff vs main    │
                │  fail on any hand-edit drift             │
                └──────────────────────────────────────────┘
                        ↓ passes
                ┌──────────────────────────────────────────┐
                │  wasm-bench        (always)              │
                │  candidate role: benchstat vs baseline   │
                │  fail on >5% regression                  │
                └──────────────────────────────────────────┘
                        ↓ passes
                ┌──────────────────────────────────────────┐
                │  z15-bench         (bench-z15 label)     │
                │  SSH to real IBM z15 LPAR, s390x bench   │
                │  fail on >5% regression on real HW       │
                └──────────────────────────────────────────┘
```

The three workflows are independent — any one can be omitted, and they
compose cleanly. A minimal consumer ships only `wasm-drift`; a
performance-sensitive consumer adds `wasm-bench`; a consumer that cares
about a specific arch's dispatch quality adds a per-arch bench with a
label gate that keeps the shared runner from being flooded.

## wasm-drift — enforce generator = committed source

The kernel source of truth is the pinned generator. The drift workflow
regenerates it and diffs vs the committed `.wat` / `.wasm`. Any hand-
edit fails CI immediately.

Consumer's `generate.go`, as used by `matchlen-wasm`:

```go
//go:generate sh -c "go run github.com/go-asmgen/asmgen/examples/wasm/matchlen@v0.6.0 > matchlen.wat"

package matchlenwasm
```

Consumer's `.github/workflows/wasm-drift.yml`:

```yaml
name: wasm-drift

on:
  push:
    branches: [main]
  pull_request:
    paths:
      - 'matchlen.wat'
      - 'matchlen.wasm'
      - 'generate.go'
      - 'Makefile'
      - '.github/workflows/wasm-drift.yml'

jobs:
  drift:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: 'stable'
      - name: Install WABT
        run: sudo apt-get update && sudo apt-get install -y wabt
      - name: Regenerate + rebuild
        run: |
          cp matchlen.wat  matchlen.wat.committed
          cp matchlen.wasm matchlen.wasm.committed
          go generate ./...
          wat2wasm matchlen.wat -o matchlen.wasm
      - name: Check for drift
        run: |
          diff -u matchlen.wat.committed matchlen.wat || {
            echo "::error::matchlen.wat drifted from pinned generator"
            exit 1
          }
          diff -q matchlen.wasm.committed matchlen.wasm || {
            echo "::error::matchlen.wasm not rebuilt from matchlen.wat"
            exit 1
          }
```

Trip-up: someone hand-edits `matchlen.wat` to add a comment. Regen wipes
the comment, diff fails, PR is blocked with an actionable error message.

## wasm-bench — portable regression detection under wazero

`wasm-bench` runs the wazero-hosted benchmark on stock ubuntu-24.04 —
cheap enough (~30s per size × 22 benchmarks × 6 count = ~3 min) to fire
on every PR without a gate.

The workflow splits into two roles by trigger:

- **BASELINE** on push-to-main / schedule / dispatch: publishes
  `bench.out` as artifact `wasm-baseline` (90-day retention).
- **CANDIDATE** on `pull_request`: downloads the freshest main-branch
  baseline via `gh` CLI, runs the same bench, `benchstat`s the two,
  posts the diff table as an upserting PR comment (marker-based so
  re-runs replace the previous comment), and fails on any benchmark
  regressed beyond `REGRESSION_THRESHOLD_PCT` (default 5%).

### The awk regression scanner

The tricky part is parsing modern `benchstat -format text` output — it
uses Unicode box-drawing characters, strips the `Benchmark` prefix, and
prints `~` on statistically insignificant deltas so noise self-drops.
The scanner tracks the current section header (`sec/op`, `B/s`,
`allocs/op`) and applies polarity per section — `sec/op +N%` is a
regression, `B/s -N%` is a regression, small speedups on either side
are ignored.

```awk
/sec\/op/          { section="time"; next }
/B\/s/             { section="throughput"; next }
/allocs\/op|B\/op/ { section="alloc"; next }
/^goos:|^goarch:|^pkg:|^cpu:/ { next }
/^[[:space:]]*$/               { next }
/^[[:space:]]*│/               { next }
{
  for (i=1; i<=NF; i++) {
    if (match($i, /^\+[0-9]+(\.[0-9]+)?%$/)) {
      val = substr($i, 2, length($i)-2) + 0
      if (section=="time" && val > thr)
        { print $1, $i; found=1 }
    }
    if (match($i, /^-[0-9]+(\.[0-9]+)?%$/)) {
      val = substr($i, 2, length($i)-2) + 0
      if (section=="throughput" && val > thr)
        { print $1, $i, "(throughput)"; found=1 }
    }
  }
}
END { exit !found }
```

Gotchas the scanner encodes:

- Modern benchstat strips the `Benchmark` prefix — anchoring on
  `^Benchmark` would silently match nothing.
- Insignificant deltas print as `~ (p=0.234 n=6)`; the signed-percent
  regex doesn't match `~`, so noise drops automatically.
- The B/s section's polarity is inverted — a `-N%` means slower, and
  the section-tracker steers the scanner into the right test.
- Everything after `allocs/op` / `B/op` is memory data, not perf —
  section resets to `alloc` and no rows there are flagged.

## z15-bench — arch-native regression detection on real hardware

`z15-bench` SSHes into a real IBM z15 LPAR (`linux1@148.100.85.193`)
via `webfactory/ssh-agent`, runs the s390x bench there, and applies the
same baseline/candidate + benchstat pattern. The one addition is a
**label gate** — the shared LPAR would drown under one-bench-per-PR
traffic, so a PR only triggers the actual SSH bench when a maintainer
has stuck the `bench-z15` label on it.

Gate structure:

```yaml
jobs:
  gate:
    runs-on: ubuntu-24.04
    outputs:
      run: ${{ steps.decide.outputs.run }}
      role: ${{ steps.decide.outputs.role }}
    steps:
      - id: decide
        run: |
          case "${{ github.event_name }}" in
            pull_request)
              if [ "${{ contains(github.event.pull_request.labels.*.name,
                                 'bench-z15') }}" = "true" ]; then
                echo "run=true"  >> $GITHUB_OUTPUT
                echo "role=candidate" >> $GITHUB_OUTPUT
              else
                echo "run=false" >> $GITHUB_OUTPUT
              fi
              ;;
            *) echo "run=true; role=baseline" >> $GITHUB_OUTPUT ;;
          esac

  bench:
    needs: gate
    if: needs.gate.outputs.run == 'true'
    # ... rest of the bench job
```

A PR without the label sees `gate` pass in 5 seconds and `bench` skip —
the LPAR is untouched.

## Proof: end-to-end validation

The reference consumer's pipeline was validated with two self-check PRs:

**PR #1 — positive path.** Docstring-only edit to `matchlen_wasm.go` (no
`.wat` / `.wasm` change). Expected verdict: pass.

- `wasm-drift`: correctly didn't fire — its `paths` filter doesn't
  include the Go wrapper.
- `wasm-bench`: ran the candidate role, downloaded main-branch
  baseline, benchstat compared 22 benchmarks × 4 sections. One
  statistically-significant delta (`MatchlenWasm/8B-4 -0.74%, p=0.041`)
  — well under 5%. Verdict: `✅ No regression above the 5% threshold`.
- `z15-bench`: label gate skipped the SSH bench (`gate` job 5s, `bench`
  job 0s / skipped) — LPAR untouched.

**PR #2 — negative path.** 50 pointless multiplications injected into
the wasm bench inner loop (~50 ns/iter overhead). Expected verdict:
fail.

- `wasm-bench`: candidate role ran the same bench harness on the PR
  head; benchstat correctly showed 8 regressed `MatchlenWasm` rows
  above threshold:

  | Benchmark | Slowdown | p-value |
  |---|---|---|
  | `MatchlenWasm/8B-4`    | +19.92% | 0.002 |
  | `MatchlenWasm/16B-4`   | +24.02% | 0.002 |
  | `MatchlenWasm/32B-4`   | +22.69% | 0.002 |
  | `MatchlenWasm/64B-4`   | +21.71% | 0.002 |
  | `MatchlenWasm/128B-4`  | +19.92% | 0.002 |
  | `MatchlenWasm/256B-4`  | +16.49% | 0.002 |
  | `MatchlenWasm/1KiB-4`  | +7.86%  | 0.002 |

  Correctly NOT flagged: `MatchlenWasm/4KiB-4 (+2.70%)` — under 5%
  threshold. Correctly NOT flagged: `16KiB..1MiB` — 50 ns dwarfed by
  µs-scale baselines so no significant delta. Correctly NOT flagged:
  `MatchlenScalar` speedups (`-0.28%`, `-0.16%`) — negative deltas on
  sec/op are speedups, not regressions.

  Verdict: `❌ One or more benchmarks regressed beyond the 5%
  threshold`. `Fail on regression` step exited non-zero → whole job
  went red.

Between the two PRs the pipeline demonstrated **no false positives**
(PR #1) and **no false negatives** (PR #2), and honored the 5%
threshold exactly. The runs are on record at
[go-simd/matchlen-wasm/pull/1](https://github.com/go-simd/matchlen-wasm/pull/1)
and
[/pull/2](https://github.com/go-simd/matchlen-wasm/pull/2).

## Adopting the pattern

1. Copy `generate.go` from `matchlen-wasm` and update the kernel name.
2. Copy `.github/workflows/wasm-drift.yml` — adjust the `paths` list to
   match your kernel's asset names.
3. Copy `.github/workflows/wasm-bench.yml` — adjust the `paths` list
   and the bench command in the `Run wazero-hosted matchlen bench` step
   to point at your kernel package.
4. Optionally copy `.github/workflows/z15-bench.yml` — same shape,
   drops in as-is if you have an SSH-reachable s390x host and can add
   the `LINUX1_SSH_KEY` + `LINUX1_KNOWN_HOSTS` secrets.
5. Verify with a self-check PR that trips no change — pipeline should
   run and pass with a "no regression" comment.

The workflow files are stable across kernels; the only per-kernel edit
is the paths list. Once the reference set is copied, adding a new
kernel to a consumer is a `//go:generate` line and a bench harness.
