# MeshBench contributor manual handoff

**Status: BLOCKED on upstream PR creation.** The manual is finished, committed
and pushed to the docs fork. `MeshBench/docs` is readable and cloneable, but
neither available write route can open its PR: the GitHub connector returns
403, `Resource not accessible by integration`; the authenticated CLI's local
GitHub Guard rejects `MeshBench/docs` because its allowed target owners are
`hermes-gadget` and `gadgethd`. The guard is not bypassed or changed. An owner
must open the upstream PR or resolve that write-policy restriction. The goal
remains incomplete until the upstream PR exists and its URL replaces the
branch-only cross-link.

## Reviewable change

- Code PR: <https://github.com/MeshBench/meshbench/pull/764>, verified at head
  `d33490a96dfab60c05661b9e71f743c2568c510f`.
- Docs base: `MeshBench/docs:main`,
  `034ac41e5081a9e337ec7f4ea7473105e04d48e3`.
- Docs branch: `hermes-gadget/docs:burn/meshbench-docs-20261023`.
- Docs commit: `eb10426aadfcdd802164414015352b53f22829b6`; the fork's remote
  branch resolves to that exact SHA.
- [Review the docs diff](https://github.com/MeshBench/docs/compare/main...hermes-gadget:docs:burn/meshbench-docs-20261023).
  This is a comparison, not an opened PR. The comparison URL returns HTTP 200;
  `gh pr list --repo MeshBench/docs --head hermes-gadget:burn/meshbench-docs-20261023 --state all --json number,url,state`
  returns `[]`.
- [Cross-link from PR #764](https://github.com/MeshBench/meshbench/pull/764#issuecomment-6365017822).
  The posted comment links the immutable docs commit and comparison, records
  verification and states the upstream-PR blocker. Its body is read back through
  the API after posting.

Only `pages/quality-gates.md` changes in the docs branch: 149 insertions and
9 deletions. It extends the existing **What the build enforces** manual,
already present in the site's navigation and overview, rather than adding a
changelog or editing generated references.

The page explains the per-entry-point CPU twin and `func(*testing.T)` callback
registration, actual coverage map entries, recursive WGSL inventory, stale
and invalid registrations, test commands, exact guard messages and device
skip/setup-failure distinction. It describes the ratchet's exit-status,
total/per-linter-summary and baseline checks, zero allowances, exact messages
and recovery steps. It states that `--update` validates the lint run but writes
counts without comparing or validating the old baseline, so it must follow a
successful comparison and cannot be used to admit a rise.

`CONTRIBUTING.md` in the main repository already links to
`https://github.com/MeshBench/docs`, the correct repository. Its cross-reference
does not need correction. No main-repository code change is required for this
manual task.

## Committed fallback artifacts

- [Finished replacement Markdown](20261023-meshbench-docs/quality-gates.md),
  byte-identical to the docs commit's `pages/quality-gates.md`.
- [Exact proposed PR body](20261023-meshbench-docs/pr-body.md).
- Proposed title: **Document GPU CPU twins and fail-closed lint checks**.
- Base/head: `MeshBench/docs:main` /
  `hermes-gadget:burn/meshbench-docs-20261023`.
- Handoff branch: `hermes-gadget/meshbench:burn/meshbench-docs-20261023`.
  [Finished page](https://github.com/hermes-gadget/meshbench/blob/burn/meshbench-docs-20261023/docs/reviews/20261023-meshbench-docs/quality-gates.md)
  and [exact PR body](https://github.com/hermes-gadget/meshbench/blob/burn/meshbench-docs-20261023/docs/reviews/20261023-meshbench-docs/pr-body.md)
  are available together on the permitted fork for the owner to submit.

The read-only repository is reachable; these artifacts preserve the finished
submission because upstream PR creation is inaccessible. They do not stand in
for a claim that the requested upstream PR has been opened.

The attempted creation command is:

```bash
gh pr create --repo MeshBench/docs --base main \
  --head hermes-gadget:burn/meshbench-docs-20261023 \
  --title 'Document GPU CPU twins and fail-closed lint checks' \
  --body-file /home/ben/worktrees/meshbench/docs-20261023/dist/meshbench-docs/pr-body.md
```

The CLI rejects that target before creation. After an owner opens the PR,
replace the branch-only cross-link on #764 with the upstream docs PR URL.

## Verification

The docs checkout lives at `dist/meshbench-docs/site` in this worktree. Code
verification uses a `git archive` of the exact PR head under
`dist/meshbench-docs/pr764`; the working branch remains
`burn/meshbench-docs-20261023`. Build outputs, toolchain, caches and logs stay
under the ignored `dist/meshbench-docs/` directory.

| command or check | observed result |
|---|---|
| `python3 gen.py` in the docs checkout | Exit 0, 41 pages. The generator checks internal links, anchors and duplicate heading IDs. |
| Shell bodies of the Pages workflow's `Every page in the navigation exists` and `No page links to a file that is not there` steps | Exit 0; all navigation targets, linked pages and referenced images exist. |
| Decode each rendered `<pre><code>` block, strip highlighting and compare with its Markdown fence | Exit 0; all six blocks match exactly, including copyable test commands. |
| Check added prose for em-dashes and review its register | Present-tense manual; no dates, first-person history or em-dashes in the added prose. |
| `git diff --check` | Exit 0 in both repositories. |
| `cmp` of both fallback files against their original finished files | Exit 0 for the manual and PR body. |
| `go test -p=2 internal/lintgate_test.go -run '^TestLintRatchetFailsClosed$' -count=1 -v` in the PR archive | Exit 0; all 15 fault-injection cases pass. |
| Targeted package command below, in the PR archive | Exit 0 for both packages; inventory, negative fixtures and CPU-only comparison guards pass. Five device comparisons skip. |

```bash
go test ./internal ./internal/rf/gpu \
  -run 'TestLintRatchetFailsClosed|TestEveryKernelHasACPUTwin|TestKernelTwinGuard|TestKernelTwins|TestLossComparison|TestCompareFold' \
  -count=1 -v
```

The host has no `go` on its initial PATH. Go 1.25.0 is installed only under
`dist/meshbench-docs/go`, with its archive SHA-256 verified against the official
`https://go.dev/dl/?mode=json&include=all` release record. The successful test
commands use that binary on PATH, `GOTOOLCHAIN=local`, and worktree-local
`GOCACHE`/`GOMODCACHE`. Logs are `ratchet-tests.log` and `targeted-tests.log`
under `dist/meshbench-docs/`.

The five `TestKernelTwins` subtests report `no GPU available:` because
llvmpipe is a software rasteriser. Real-device numerical agreement is not
verified here. No full application test suite or actual golangci-lint run is
claimed for this documentation change.

The generated HTML is inspected directly, including section anchors,
ordered-step continuity, message tables and code-copy fidelity. The preview
serves HTTP 200 and loads its local fonts. The in-app Browser cannot connect
because its trusted browser service is unavailable; separate headless capture
attempts produce no screenshot, with the bounded attempt exiting 124. Visual
screenshot QA is therefore not claimed.

## Completion audit

The full brief is read. The docs checkout and contributor/CI page are found;
all required contract content, exact commands and failure messages are in the
finished manual. The docs change is committed and pushed, targeted build and
tests are verified, and the #764 cross-link is posted and read back. The
finished Markdown, exact PR body and this required handoff are committed on
the requested worktree branch. Upstream PR creation remains blocked by the
recorded write restrictions; there is no claim of a docs PR URL or full goal
completion.
