MeshBench contributors need the CPU-twin registration and lint failure contract when working on GPU kernels or resolving CI findings. This is the companion manual change for [MeshBench/meshbench#764](https://github.com/MeshBench/meshbench/pull/764).

Extend **What the build enforces** with ordered registration steps, actual CPU function and comparison callback references, the recursive WGSL inventory, targeted test commands, and the distinction between unavailable-device skips and setup failures. Document the ratchet's output/status/baseline validation, per-linter allowances, exact diagnostic messages and recovery steps, including the limits of `--update`. The page uses present-tense manual prose; no generated references or build configuration change.

Validation:

- `python3 gen.py`: passes, 41 pages; internal links, anchors and heading IDs validated by the generator.
- The Pages workflow's navigation, page-link and image-existence checks: pass.
- All six code blocks match their rendered and copyable text exactly; `git diff --check` passes.
- Against code PR head `d33490a96dfab60c05661b9e71f743c2568c510f`, `go test ./internal ./internal/rf/gpu -run 'TestLintRatchetFailsClosed|TestEveryKernelHasACPUTwin|TestKernelTwinGuard|TestKernelTwins|TestLossComparison|TestCompareFold' -count=1 -v`: passes with Go 1.25.0, including all 15 ratchet fault-injection cases and the CPU-only kernel/comparison guards. The five device comparison subtests skip because the host offers only llvmpipe; no real-device numerical parity is claimed.
