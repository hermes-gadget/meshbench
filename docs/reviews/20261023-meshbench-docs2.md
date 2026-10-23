# MeshBench CPU-twin and lint-ratchet manual

**Status: BLOCKED on upstream PR creation; the brief's committed fallback is delivered.**
The separate `MeshBench/docs` repository is cloned inside this worktree at
`dist/meshbench-docs`. Its finished manual is committed and pushed to the
docs fork. Direct upstream push returns HTTP 403, upstream PR creation through
the GitHub integration returns 403, and the automatic GitHub Guard rejects
the CLI PR target. No upstream docs PR exists. No guard or configuration is
changed or bypassed.

## Reviewable artifacts

- Code PR: <https://github.com/MeshBench/meshbench/pull/764>, open at
  `d33490a96dfab60c05661b9e71f743c2568c510f`.
- Docs base: `MeshBench/docs:main`,
  `034ac41e5081a9e337ec7f4ea7473105e04d48e3`.
- Docs branch: `hermes-gadget/docs:burn/meshbench-docs2-20261023`.
- Docs commit: `81ef4cb84e495eae815f2ffb84e93d9429612fa3`, verified against
  the fork's remote branch.
- [Finished docs page](https://github.com/hermes-gadget/docs/blob/81ef4cb84e495eae815f2ffb84e93d9429612fa3/pages/quality-gates.md).
- [Docs comparison](https://github.com/MeshBench/docs/compare/main...hermes-gadget:docs:burn/meshbench-docs2-20261023).
  This comparison is not an opened PR.
- [Cross-link comment on #764](https://github.com/MeshBench/meshbench/pull/764#issuecomment-6367237370),
  posted using the authenticated CLI after the connector refuses comment
  creation, then read back through the API and compared with the submitted body.
- [Committed fallback Markdown](20261023-meshbench-docs2/quality-gates.md)
  and [exact proposed PR body](20261023-meshbench-docs2/pr-body.md).
  Both are committed in `hermes-gadget/meshbench` at
  `5c941898679bbae380e3e10502af05c6c55c30ef` and pushed on the requested
  `burn/meshbench-docs2-20261023` branch.

Only `pages/quality-gates.md` changes in the docs repository: 163 insertions
and 9 deletions. The existing **What the build enforces** page remains in the
site navigation. The added text uses present tense, third person and British
English, with no dates or em-dashes.

The manual covers CPU-function and `func(*testing.T)` callback registration
for each compute entry point, including both entry points in
`coverage.wgsl`; recursive inventory of nested and unembedded WGSL; stale,
missing, duplicate and invalid registrations; the actual comparison callback
and its assertions; exact commands and guard diagnostics; unavailable-device
skips versus shader/pipeline failures; and the distinction between source
inventory and hardware numerical agreement.

The ratchet section covers per-linter allowances, count-based limitations,
exit-status and summary consistency, baseline format, all script messages,
the pinned linter version, executable overrides and recovery steps.
It explicitly states that `--update` validates the lint run but neither
validates nor compares the old baseline, so it must follow a successful
comparison and must not admit a rise.

## Verification

The code checks use a `git archive` of the exact #764 head under
`dist/pr764`; the worktree's source remains at its original base.
Go 1.25.0 is read from the existing local toolchain at
`/home/ben/worktrees/meshbench/docs-20261023/dist/meshbench-docs/go/bin`.
That directory is added to PATH; `GOTOOLCHAIN=local`, `GOENV=off` and
`GOMAXPROCS=2` are set. `GOCACHE`, `GOMODCACHE` and `GOTMPDIR` all
point inside this worktree's ignored `dist/`. No toolchain or cache in
another worktree is modified.

| exact command or check | observed result |
|---|---|
| `python3 gen.py` in `dist/meshbench-docs` | Exit 0; 41 pages; generator checks links, anchors and duplicate heading IDs. |
| Unmodified shell body of Pages workflow step `Every page in the navigation exists` | Exit 0. |
| Unmodified shell body of Pages workflow step `No page links to a file that is not there` | Exit 0; linked pages and referenced images exist. |
| Decode generated `<pre><code>` blocks and compare with Markdown fences | Exit 0; all six blocks match exactly. |
| Parse rendered ordered lists and inspect contract headings | Exit 0; three lint steps and five kernel steps remain intact. |
| `curl --silent --fail http://127.0.0.1:8764/quality-gates.html -o dist/served-quality-gates.html`, then `cmp` with generated HTML | Exit 0; the local preview serves the built page. |
| `go test ./internal/rf/gpu -run '^TestEveryKernelHasACPUTwin$' -count=1 -v` | Exit 0; source inventory passes. |
| `go test ./internal/rf/gpu -run '^TestKernelTwins$' -count=1 -v` | Exit 0; all five device subtests skip on llvmpipe. |
| `go test ./internal ./internal/rf/gpu -run 'TestLintRatchetFailsClosed\|TestKernelTwinGuard' -count=1 -v` | Exit 0; all 15 ratchet cases and CPU-only kernel guard fixtures pass. |
| `go test ./internal/rf/gpu -run 'TestLossComparison\|TestCompareFold' -count=1 -v` | Exit 0; invalid-readback and divergence guards pass. |
| `git diff --check` in both repositories | Exit 0. |
| `cmp docs/reviews/20261023-meshbench-docs2/quality-gates.md dist/meshbench-docs/pages/quality-gates.md` | Exit 0; fallback page is byte-identical to the docs commit. |
| `git ls-remote fork refs/heads/burn/meshbench-docs2-20261023` in the docs clone | Resolves to `81ef4cb84e495eae815f2ffb84e93d9429612fa3`. |
| `gh pr list --repo MeshBench/docs --head hermes-gadget:burn/meshbench-docs2-20261023 --state all --json number,url,state` | Exit 0; `[]`, confirming no upstream PR exists for this branch. |

Logs remain under ignored `dist/`: `docs-build.log`,
`inventory-tests.log`, `comparison-tests.log`, `negative-tests.log`
and `comparison-guard-tests.log`. The Pages workflow shell bodies are read
from the cloned repository and executed without modification.

No real-device numerical parity is claimed. The comparison subtests report
`no GPU available:` because llvmpipe is a software rasteriser. No full
application suite or actual `golangci-lint run` is claimed for this manual-only
change. The Browser skill's preview attempt reports
`Browser use requires a trusted Node REPL browser service`; generated
HTML, code-copy fidelity and local HTTP serving are checked, but visual
screenshot QA is unavailable.

## Exact upstream refusal and submission

The permitted direct push command is:

```bash
git push origin HEAD:refs/heads/burn/meshbench-docs2-20261023
```

It exits 128:

```text
remote: Permission to MeshBench/docs.git denied to hermes-gadget.
fatal: unable to access 'https://github.com/MeshBench/docs.git/': The requested URL returned error: 403
```

The GitHub connector's `create_pull_request` call for
`MeshBench/docs`, base `main`, head
`hermes-gadget:burn/meshbench-docs2-20261023` returns:
`GitHub API error 403: Resource not accessible by integration`.

The CLI request uses the committed body file:

```bash
gh pr create --repo MeshBench/docs --base main \
  --head hermes-gadget:burn/meshbench-docs2-20261023 \
  --title 'Document CPU twins and fail-closed lint checks' \
  --body-file docs/reviews/20261023-meshbench-docs2/pr-body.md
```

It exits 1:

```text
GITHUB GUARD: PR creation blocked
  allowed owners: hermes-gadget gadgethd
  attempted target: MeshBench/docs
  target source: --repo
```

An owner with an authorised upstream write route can open the PR from the
already pushed docs branch using the title and exact body below, then add its
URL to #764. The current comment links the immutable docs commit and fallback
artifacts, and states the PR-creation blocker.

**Title:** Document CPU twins and fail-closed lint checks

**Base:** `MeshBench/docs:main`

**Head:** `hermes-gadget:burn/meshbench-docs2-20261023`

**Exact proposed PR body:**

```markdown
Contributors adding a GPU compute entry point need a CPU function and a comparison callback that the test registry executes. The lint ratchet also needs a completed, consistent lint run before it can compare or write counts.

Extend the existing **What the build enforces** manual with:

- Per-entry-point registration, comparison callback steps, recursive WGSL inventory checks and exact guard diagnostics.
- Source-checkout test commands, device-unavailable skips versus shader/pipeline failures, and the limits of a parent `PASS` when every device subtest skips.
- Linter exit-status, summary and baseline validation; per-linter allowances; exact refusal messages and recovery steps.
- The safe baseline-tightening sequence and the fact that `--update` writes current counts without comparing or validating the old baseline.

Companion manual for https://github.com/MeshBench/meshbench/pull/764. Only `pages/quality-gates.md` changes; generated references, navigation and workflows stay as they are.

Validation:

- `python3 gen.py`: exit 0, 41 pages; internal links, anchors and heading IDs checked.
- Both Pages-workflow navigation/link/image checks: exit 0.
- All six rendered code blocks match the Markdown exactly; ordered lists retain all three lint steps and five kernel steps.
- On code PR head `d33490a96dfab60c05661b9e71f743c2568c510f`, the documented `TestEveryKernelHasACPUTwin` command passes. `TestKernelTwins` exits 0 with all five device subtests skipped on llvmpipe; hardware numerical parity is not verified.
- `go test ./internal ./internal/rf/gpu -run 'TestLintRatchetFailsClosed|TestKernelTwinGuard' -count=1 -v`: exit 0; all 15 ratchet fault-injection cases and the CPU-only kernel guards pass.
- `go test ./internal/rf/gpu -run 'TestLossComparison|TestCompareFold' -count=1 -v`: exit 0; invalid-readback and divergence guards pass.
- `git diff --check`: exit 0.

The local preview serves the generated page correctly. The Browser service is unavailable, so visual screenshot QA is not claimed.
```

## Completion audit

The full brief is read. The correct separate repository is cloned; the
manual is grounded in the exact code PR head, built, committed and pushed to
the docs fork. Its finished Markdown and exact PR body are committed in this
worktree as required by the verified upstream-push refusal. The #764 comment
is posted and read back. This required handoff is committed and pushed on the
requested branch. No services, live configuration, secrets, workflows,
fixtures, dependencies or application source are changed; no agents are
spawned. Work finishes before the 16:15 BST hard stop.

The requested upstream docs PR remains uncreated because both available
creation routes refuse it. The fallback is delivered; opening and cross-linking
the upstream PR still requires an authorised external action.
