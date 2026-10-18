# Session handoff — 2026-10-14 / next: merge latest upstream into `main`

Read **this file first**, then `docs/MERGE-UPSTREAM-2026-10-14.md` (the completed v0.15.0 merge) if you need conflict history.

## Repo layout (this is the source of truth)

| | |
| --- | --- |
| Working tree | `/home/ugo/projects/gufo` |
| **Development branch** | **`main`** (you are on it) |
| **Fork remote** | **`uv-genai`** → `https://github.com/uv-genai/gufo.git` — **default branch `main`** |
| **Upstream remote** | **`origin`** → `https://github.com/gufo-org/gufo.git` |
| **Upstream tracking branch** | local **`gufo-org`** → `origin/main` (upstream `main`) |
| Deleted | local/remote `uv-rdma` (merged into fork `main`) |

Current tips (as of handoff):

- fork `main` / `uv-genai/main`: **`f84703b1`** `Thunderbolt  rdma setup`
- upstream `gufo-org` / `origin/main`: **`51a8b00d`** `perf(qwen-flash): score MTP drafts over request vocabulary (#603)` — **this is ahead of what was merged** (v0.15.0 merge was `89d61855`)

## Already done (do not redo)

1. Merged upstream **v0.15.0** (`89d61855`) into what was then `uv-rdma` → commit **`024abbb7`**, tag **`tp2-rdma-merge-2026-10-14`**.
2. TP2 correctness fixes after that merge (see architecture rules in project `MEMORY.md`):
   - `defer_residual_` only when `tp_world_size()==1`
   - `TpMirroredRunner::UsesComponentCache()==false` (mirrored session restore)
   - `PrefillThrough` → `Prefill`; `in_pass_checkpoint=!distributed_`
   - standalone `CreateState()` for TP2 executor
3. Launchers: `tools/tp2/` (rank0/1 q4/q8, serve-q4, swap), `tools/llama-swap/` (single-q4.sh, tp2.sh).
4. Benchmarks saved: `docs/models/qwen3.8-flash-next/2026-10-14-tp2-depth-series.md` + artifacts. Q4/Q8 TP2 beats matched one-host.
5. Docs/README/AGENTS updated (AGENTS **0× nix**; README native-only build).
6. Promoted fork **`main`** as default on `uv-genai`; deleted `uv-rdma`.
7. GitHub issue: `uv-genai/gufo#1` — YaRN/rope scaling feature request (user’s repo only).

## Next task (what was about to start)

**Merge latest upstream into fork `main`:**

```sh
# 1. refresh upstream tracking
git fetch origin
git switch gufo-org && git merge --ff-only origin/main   # or reset to origin/main

# 2. inspect delta
git log --oneline main..gufo-org
git diff --stat main...gufo-org

# 3. merge into main (on main)
git switch main
git merge gufo-org   # resolve conflicts like 2026-10-14 merge
```

Expect more work in the same files as last time if upstream touched them:
`kernels.hip.cpp`, `executor.cpp`, `device_model.cpp`, `inference_backend.cpp`.

**Non-negotiables (same as last merge):**

1. `grep -ci nix AGENTS.md` must stay **0**. Never run `nix`.
2. Build: native CachyOS flags (see project MEMORY / AGENTS.md).
3. After merge: `cmake --build --preset release`, `./build/release/gufo diagnose`.
4. **TP2 check with generation text**, not HTTP 200. Prompt **>1024 tokens**.
5. Before any GPU launch: **no `gufo` on either host**; ship binary to
   `strix-halo-1.local:/home/ugo/projects/gufo/build/release/gufo`.
6. Only one `gufo` per GPU host. Launchers: `tools/tp2/`.
7. Push **only** to `uv-genai` (user-owned). **Never** write issues/PRs on `gufo-org`.

**TP2 pair (when validating):**

- rank 0: `strix-halo-1` `192.168.1.22` — `tools/tp2/rank0-q4.sh 8000`
- rank 1: `strix-halo-2` `192.168.1.92` (this box) — `tools/tp2/rank1-q4.sh`
- Start rank 0, then rank 1 within 30 s (detached `ssh -f` / `setsid nohup`).
- Ready: `curl http://192.168.1.22:8000/ready`

## Key files

| path | purpose |
| --- | --- |
| `MEMORY.md` (project memory) | hard rules, TP2 architecture |
| `docs/MERGE-UPSTREAM-2026-10-14.md` | last merge write-up |
| `docs/models/qwen3.8-flash-next/2026-10-14-tp2-depth-series.md` | benchmarks |
| `/home/ugo/gufo-tp2-rdma-merge-2026-10-14.md` | session export |
| `docs/models/qwen3.8-flash-next/TP2.md` | protocol / qualification |
| `tools/tp2/`, `tools/llama-swap/` | launchers |

## Known open items

- YaRN/context: Flash-Next native 262K only; issue `uv-genai/gufo#1`.
- Q8 concurrent only tested to 4 users (`sessions=4` at 262K).
- Broader tree still mentions Nix (`flake.nix`, `TESTING.md`, …) — strip only if asked.
