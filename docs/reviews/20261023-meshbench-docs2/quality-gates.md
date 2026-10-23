# What the build enforces

MeshBench's limits are mechanical rather than advisory, because taste does not
survive scale. This is what actually fails a build, what only reports, and why
each one exists.

<figure>
<svg viewBox="0 0 780 240" role="img" aria-label="Which checks run at which trigger">
  <text x="20" y="30" font-size="12" font-weight="600" fill="var(--ink)">every pull request</text>
  <rect x="20" y="40" width="740" height="44" rx="8" fill="var(--good)" fill-opacity=".08" stroke="var(--good)" stroke-opacity=".4"/>
  <text x="390" y="67" font-size="11.5" fill="var(--ink)" text-anchor="middle">gofmt &#183; go vet &#183; golangci-lint + ratchet &#183; tests, four shards &#183; conflict markers &#183; file limits &#183; layout map &#183; licence inventory</text>
  <text x="20" y="116" font-size="12" font-weight="600" fill="var(--ink)">a v* tag, or a deliberate dispatch</text>
  <rect x="20" y="126" width="740" height="44" rx="8" fill="var(--warn)" fill-opacity=".08" stroke="var(--warn)" stroke-opacity=".4"/>
  <text x="390" y="153" font-size="11.5" fill="var(--ink)" text-anchor="middle">the race detector &#183; the release pipeline (package.yml) - nothing in a pull request exercises either</text>
  <text x="20" y="202" font-size="12" font-weight="600" fill="var(--ink)">on demand, for reading rather than gating</text>
  <rect x="20" y="212" width="740" height="24" rx="8" fill="var(--panel)" stroke="var(--rule)"/>
  <text x="390" y="228" font-size="11.5" fill="var(--dim)" text-anchor="middle">SonarQube: complexity, whole-tree duplication, per-package coverage</text>
</svg>
<figcaption>A green pull request is evidence about the first lane only. The
second lane stays invisible until a tag or a dispatch fires it.</figcaption>
</figure>

## On every pull request

| gate | what it refuses |
|---|---|
| `gofmt -l` | any file not formatted |
| `go vet` | the compiler's own suspicions |
| `golangci-lint` + ratchet | an increased per-linter count, incomplete lint or an invalid baseline |
| `go test ./...` | a failing test, across four parallel shards |
| GPU CPU twins | a compute entry point without a CPU function and comparison callback |
| conflict markers | `<<<<<<<`, `=======` or `>>>>>>>` committed in source |
| file length | over 500 lines without an exemption naming its reason |
| tracked build artifacts | a compiled binary or coverage profile in the index |
| the layout map | a package with no entry, or an entry with no package |
| the licence inventory | a dependency change that has not regenerated it |

The last three exist because each failure they refuse arrives silently and
looks fine in review.

## The lint ratchet

Twenty linters run, not a token few and not all 104. The
enabled set is those whose findings are worth acting on here, each with its
reason recorded in `.golangci.yml`.

Enabling the bug-class linters on an existing tree produces hundreds of
findings at once. Turning those into a red build would mean hundreds of
unreviewed changes in one commit, so the count is held at a baseline instead:

```bash
tools/lint-ratchet.sh
```

Run the command from a MeshBench source checkout. It invokes
`golangci-lint run` and compares each linter's count against
`.golangci-baseline.txt`. A linter absent from the baseline has an allowance
of zero. An increased count fails even if another linter's count falls; an
equal or lower count passes. This is a count check: replacing a finding with
another from the same linter can leave the count unchanged.

The script needs Bash and the source checkout's lint configuration. Running
the documentation site's `python3 gen.py` checks the manual's links and
anchors; it does not run the application's lint or GPU tests.

The ratchet fails closed. Before it compares or writes counts, it requires:

- A linter exit status of 0 for zero findings, or 1 for findings. A missing
  executable, configuration error, timeout or crash is not a clean run.
- Exactly one recognised total line, `N issues.` or `N issues:`, with a
  non-negative integer in place of `N`.
- Per-linter summary lines in the form `* gosec: 2`. Names start with a
  lower-case letter and contain only lower-case letters and digits. Their
  counts must sum to the total, and the total must agree with the exit status.

In comparison mode, the baseline must also be readable. Each non-blank line
contains a linter name in that same form and a non-negative integer count,
separated by whitespace; duplicate names and extra fields fail. An empty
baseline allows no findings.

### Fix findings and tighten the baseline

1. Use the version named by `GOLANGCI_LINT_VERSION` in
   `.github/workflows/ci.yml` (v2.12.2). Different versions disagree about the
   tree by tens of findings, so a baseline from another version is not a
   meaningful comparison. `golangci-lint version` prints the installed version.
2. Run `tools/lint-ratchet.sh`. Fix each increased count. A local
   `//nolint` exception must name the linter and give the reason at the call
   site; raising the allowance does not resolve the finding.
3. After a successful comparison reports `fallen` or `cleared`, record the
   lower counts, review the diff and check again:

```bash
tools/lint-ratchet.sh --update
git diff -- .golangci-baseline.txt
tools/lint-ratchet.sh
```

Commit the tightened baseline with the fixes that earn it. `--update` checks
the lint run, then writes its counts directly: it neither validates nor
compares the old baseline, and can therefore raise an allowance if used
without the preceding comparison. Never use it to admit an increased count.
A failed or unrecognised lint run leaves the baseline unchanged, including
when `--update` is requested.

`GOLANGCI_LINT` can select a different executable path; its value names one
executable, not a command with extra arguments:

```bash
GOLANGCI_LINT=/path/to/golangci-lint tools/lint-ratchet.sh
```

### Ratchet messages

The placeholders below stand for the linter name, counts and exit status.
The spelling and capitalisation match the script.

| message | meaning and action |
|---|---|
| `lint did not complete (exit <status>); baseline unchanged` | The linter exits with a status other than 0 or 1. Read the preceding tool output, restore the executable or fix the tool failure, then rerun. |
| `unrecognised lint output; baseline unchanged` | There is no single recognised total, even if the linter exits successfully. Check the pinned version and the preceding output. |
| `inconsistent lint summary; baseline unchanged` | Per-linter counts do not sum to the total, or the total disagrees with the exit status. Resolve the tool/output mismatch before updating. |
| `invalid or missing lint baseline` | Comparison mode cannot read the baseline, or its rows are malformed or duplicated. Restore the tracked baseline or correct its format without widening it. |
| `RISEN  <linter>: <old> -> <count>` | This linter exceeds its allowance. Fix the findings. |
| `New lint findings. Fix them, or if they are genuinely acceptable, say why` | The failure continues on the next line: `at the call site with a //nolint comment carrying a reason.` |
| `fallen <linter>: <old> -> <count>   (run tools/lint-ratchet.sh --update)` | Some findings are resolved. After a successful comparison, record the lower count. |
| `cleared <linter>: <old> -> 0   (run tools/lint-ratchet.sh --update)` | The linter has no summary row. After a successful comparison, remove its allowance through `--update`. |
| `lint ratchet holds` | Comparison succeeds. This can still include findings already allowed by the baseline. |
| `baseline updated:` | Update succeeds; the following lines are the written counts. Review them before committing. |

Four `gosec` rules are excluded with the reason stated in the file. `G115`,
integer overflow on conversion, is the notable one: in the RF packages those
need auditing by hand rather than by rule, because a truncated sample index
produces a plausible waterfall and slightly wrong sensitivity, which no
automatic check distinguishes from a safe cast.

## Every GPU kernel has a CPU twin

Every WGSL compute entry point under `internal/rf/gpu/` has a CPU function
and a comparison callback registered in `kernelTwins`, in
`internal/rf/gpu/kernels_test.go`. Registration is per entry point, not per
file: `coverage.wgsl:main` and `coverage.wgsl:fold` need separate entries.

`TestEveryKernelHasACPUTwin` walks the source recursively, including nested
and unembedded `.wgsl` files. It ignores line and nested block comments,
checks every compute entry point against the registry, and checks the
registry back against the source. This inventory check needs no GPU.
`TestKernelTwins` checks the inventory again, then runs each registered
callback as a named subtest, so a callback named only in prose is insufficient.
The inventory checks registrations rather than WGSL validity or numerical
agreement. The comparison callback supplies the assertions, and device setup
compiles the shader and pipeline.

### Register and compare a kernel

1. Implement the CPU function and the WGSL entry point. The CPU reference
   must remain usable without a GPU.
2. Write a comparison function with signature `func(*testing.T)` in the GPU
   package's test files. Run both implementations on the same inputs and
   assert that their results agree within a justified tolerance. Check
   lengths and non-finite results explicitly; terrain comparisons also
   check that both sides agree about missing data. Existing callbacks such
   as `compareDechirpCPU` and `compareCoverageFoldCPU` show the pattern.
3. Add an entry to `kernelTwins` using the shader's path relative to
   `internal/rf/gpu/`, a colon and the entry-point name. Store actual function
   references for both the CPU implementation and the comparison.
4. Update or remove the registration when its entry point is renamed, moved
   or removed. A nil CPU value, a typed-nil function, a non-function or a nil
   comparison fails the inventory check.
5. Run the inventory and comparison tests from the MeshBench checkout.

For example, the coverage entries in the registry are:

```go
"coverage.wgsl:main": {propagation.GridLossCPU, compareCoverageCPU},
"coverage.wgsl:fold": {propagation.FoldStationCPU, compareCoverageFoldCPU},
```

The targeted test commands run from the MeshBench source checkout, with its
Go toolchain and native build dependencies available:

```bash
go test ./internal/rf/gpu -run "^TestEveryKernelHasACPUTwin$" -count=1 -v
go test ./internal/rf/gpu -run "^TestKernelTwins$" -count=1 -v
```

Device comparisons acquire a device through `comparisonDevice(t)`, which
registers cleanup with the test. If device acquisition reports unavailable
hardware, the subtest skips with `no GPU available: <reason>`. Shader or
pipeline setup failures fail with `GPU kernel setup failed: <reason>`;
comparison and readback errors fail as well. A skip supplies no evidence of
numerical agreement: run the comparisons on a supported hardware adapter
before claiming that agreement.

The registered subtests are `coverage.wgsl:fold`, `coverage.wgsl:main`,
`dechirp.wgsl:main`, `demod.wgsl:main` and `pairs.wgsl:main`. A parent
`TestKernelTwins` result can say `PASS` while all five subtests say `SKIP`.
The subtest results distinguish an inventory pass from a device comparison
that actually runs.

### Kernel inventory messages

`<path>` is relative to `internal/rf/gpu/`, and `<entry>` is the WGSL function
name. The test can report several problems together.

| message | action |
|---|---|
| `<path>:<entry> has no registered CPU twin and comparison` | Implement both functions and register the entry point. |
| `<path>:<entry>: CPU twin must be a non-nil function` | Replace the absent, typed-nil or non-function value with the CPU function reference. |
| `<path>:<entry>: missing comparison test` | Register the non-nil comparison callback. |
| `<path>:<entry>: registration has no kernel` | Correct or remove a registration whose path or entry no longer exists. |
| `<path>:<entry>: duplicate compute entry point` | Give each compute entry point a unique name within its source file. |
| `<path>: cannot identify every compute entry point` | Check the WGSL declarations: each compute attribute must resolve to an identifiable entry point, and each shader file must contain one. |
| `no GPU compute entry points found; has the source moved?` | Check the source location and inventory walker; an empty scan cannot satisfy the rule. |

The checks themselves have CPU-only negative fixtures. Run them with:

```bash
go test ./internal ./internal/rf/gpu \
  -run "TestLintRatchetFailsClosed|TestKernelTwinGuard" -count=1 -v
```

## The race detector

Not on every push - it multiplies an already slow suite by enough that the wait
becomes the thing people notice about the pipeline, and a check nobody waits for
is a check that gets worked around.

It runs when a `v*` tag is pushed, which is the moment it matters and the moment
nobody is waiting on the answer, and on request from the Actions tab. It is the
only gate that can find a startup race - exactly the class of fault review does
not catch.

## SonarQube

Run on demand rather than in CI, because a second gate to satisfy before a merge
is a second thing to work around. This one is for reading.

It covers what a per-file linter cannot measure: cognitive complexity per
function, duplication found across the whole tree rather than thresholded, and
coverage readable per package instead of as one number. Whole-tree duplication
is the class of finding a per-file linter cannot produce.

## What only reports

Some limits are stated but not enforced, and it is worth knowing which.
The house rules for the interface are one of them: they are carried as an
[agent skill](agent-skills.html) rather than as a check, because most of them
are about what a control means rather than about what compiles.

Function length is a **soft** 50 lines and 256 functions exceed it; enforcing
that would be inventing a rule rather than mechanising one. Nesting depth has
partial cover from the linter set. One type per panel file has four known
violations and wants those fixed before it can be a gate.

## The release pipeline is checked separately, and less

`package.yml` runs only on a tag push or a manual dispatch. **Nothing in a pull
request exercises it**, which is worth stating plainly: a breakage in it sits
invisible behind green pull-request checks until somebody dispatches it
deliberately.

If a change touches the release pipeline, dispatch it and read the result. A
green tick on the pull request is evidence about `ci.yml` and nothing else.
