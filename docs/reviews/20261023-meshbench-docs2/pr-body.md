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
