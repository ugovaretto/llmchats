# Session prompt — 2026-10-18 / next: TP2-qualify v0.17.0

Read **this file first**, then
`/home/ugo/gufo-merge-handoff/HANDOFF-2026-10-18.md`, then (in the repo)
`docs/MERGE-UPSTREAM-2026-10-17.md` if you need merge/conflict history.

## What is already done

- Upstream **v0.16.0** and **v0.17.0** are merged into fork `main`.
- TP2 code gates from the v0.16 hangs are in-tree (draft-subset, n-gram
  `embeddings_ready`, `OwnF16` `m==48`).
- Release **builds**. Docs (`AGENTS.md`, `README.md`, `TP2.md`) updated.
- Published **`tp2-rdma-merge-v0.17.0-untested`** on `uv-genai/main`
  (`30cb0bd7`). **TP2 runtime on v0.17.0 is not qualified yet.**

## Your job

When the user says the GPU is free (or it already is):

1. Confirm **no `gufo`** on `strix-halo-1.local` or this host.
2. Ship **this host’s** `build/release/gufo` to
   `strix-halo-1.local:/home/ugo/projects/gufo/build/release/gufo`.
   Both **sha256 must match**. Do not re-clone the remote repo; copy the
   binary (and `tools/tp2/` scripts only if they changed).
3. Start the TP2 pair with `tools/tp2/rank0-q4.sh 8000` then
   `rank1-q4.sh` within 30 s. Wait for `/ready`.
4. Qualify with **generation text** (not HTTP 200):
   - prompt **>1024 tokens**
   - **≥5 requests** at `--sessions 4` (session reuse)
   - one **repeated-prefix / agent-style** prompt (prompt lookup #613)
5. If it passes: tag `tp2-rdma-merge-v0.17.0-<date>` (no `-untested`) and
   push **only** to `uv-genai`.
6. If it hangs or returns empty text: stop both ranks; compare binary shas;
   read `TP2.md` collective rules and project memory before changing code.

## Repo layout

| | |
| --- | --- |
| Working tree | `/home/ugo/projects/gufo` |
| Development branch | **`main`** (tip `30cb0bd7`) |
| Push remote | **`uv-genai`** only |
| Upstream tracking | local **`gufo-org`** → `origin/main` — never push |

## Non-negotiables

1. `grep -ci nix AGENTS.md` stays **0**. Never run `nix`.
2. Build with the host CMake flags in `AGENTS.md` / project MEMORY.
3. One `gufo` per GPU host. Ask before any GPU launch if unsure.
4. Same git revision and binary **sha256** on both ranks.
5. Push **only** to `uv-genai`. Never write issues/PRs on `gufo-org`.
6. Do not treat a compile-only merge as TP2-qualified.

## TP2 pair

- rank 0: `strix-halo-1` `192.168.1.22` — `tools/tp2/rank0-q4.sh 8000`
- rank 1: `strix-halo-2` `192.168.1.92` (this box) — `tools/tp2/rank1-q4.sh`
- Ready: `curl http://192.168.1.22:8000/ready`
- Concurrency gauges (gufo port, not llama-swap):
  `curl -s http://192.168.1.22:8000/metrics | grep requests_`

## Key paths

| path | purpose |
| --- | --- |
| `/home/ugo/gufo-merge-handoff/HANDOFF-2026-10-18.md` | full state |
| `docs/MERGE-UPSTREAM-2026-10-17.md` | merge resolutions |
| `docs/models/qwen3.8-flash-next/TP2.md` | TP2 design + gates |
| `AGENTS.md` | build, remotes, TP2 rules |
| `tools/tp2/` | launchers |
