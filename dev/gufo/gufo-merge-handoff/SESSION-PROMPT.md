Continue an in-progress upstream merge in `/home/ugo/projects/gufo`, branch `uv-rdma`.

**Read `docs/MERGE-UPSTREAM-2026-10-14.md` first — it is the full handoff and answers nearly everything.** Then work from it.

State: `git merge` of `origin/main` (v0.15.0) is parked mid-conflict. `MERGE_HEAD` is present. 8 of 12 conflicted files are already resolved and staged; 4 remain with 10 hunks:

- `src/models/qwen38_flash_next/kernels/rocm/device_model.cpp` — 2 hunks
- `src/models/qwen38_flash_next/kernels/rocm/executor.cpp` — 5 hunks
- `src/models/qwen38_flash_next/kernels/rocm/kernels.hip.cpp` — 3 hunks

**Why this merge matters:** upstream commit `a33e989c` (`perf(qwen-flash): speed up wide prefill kernels`, PR #571) rewrote these same kernels, and this branch carries TP2/RDMA tensor-parallel changes to them.

**Non-negotiables:**

1. **Do not merge `AGENTS.md`.** It must contain zero occurrences of `nix`. Upstream didn't touch it, so it's already correct — verify with `grep -ci nix AGENTS.md` before committing and keep it at 0. Never run `nix`; this host does not have it and never will.
2. **`README.md` merged already — leave our changes untouched.** Our RDMA/TP2 note is intact. There is a pre-existing duplicated logo `<img>` tag; leave it unless asked.
3. **This is a native CachyOS build, not Nix.** Configure with:
   ```
   unset ROCM_PATH
   export PATH=/opt/rocm/bin:/opt/rocm/core/lib/llvm/bin:$PATH
   cmake --preset release -DCMAKE_HIP_COMPILER=/opt/rocm/core/lib/llvm/bin/clang++ \
     -DCMAKE_CXX_FLAGS="-include format" -DCMAKE_HIP_FLAGS="-include format"
   ```

**The hard part, and the reason care is needed:** in `device_model.cpp` upstream restructured weight loading into a two-pass measure-then-place scheme and changed `Layer(l)` to a single argument. This branch has no `Model()` — it inlines the walk and calls `Layer(l, trunk_heads_, trunk_split_)` with TP2 head splitting. That logic must move into upstream's `Model()`. **Getting it wrong silently produces wrong logits, not an error**, so any resolution here needs a TP2 two-host run to confirm, not just a build.

Also check the `half_stride` fix survives in `kernels.hip.cpp`: this branch scoped the strided `ssm_out` path to `cols == 6144` to fix `SSM output F16 GEMM failed` at >=1024-token prompts, and TP2 has `cols == 3072`. Upstream reworked those kernels. Test at least one prompt above 1024 tokens.

**Before publishing:** build, `./build/release/gufo diagnose`, and the TP2 two-host check. Do not push a merge that only compiles.

Useful background, read only if needed: `docs/models/qwen3.8-flash-next/DECODE-PERF.md` (decode profiling on this branch), `docs/models/qwen3.8-flash-next/IU4-PLAN.md` (closed IU4 study, branch `iu4`). Backup of the merge state and resolutions patch: `~/gufo-merge-handoff/`.
