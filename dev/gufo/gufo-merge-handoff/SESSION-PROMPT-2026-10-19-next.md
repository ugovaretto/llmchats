# Session prompt — 2026-10-19 / next: TP2 requalify after reboot

Read **this file first**, then
`/home/ugo/gufo-merge-handoff/HANDOFF-2026-10-19.md`, then in the repo
`docs/MERGE-CHECKLIST.md` and `docs/MERGE-UPSTREAM-2026-10-19.md`.

## Already done

- Upstream **v0.16.0 + v0.17.0** merged into fork `main` (tip **`e3fd7c5a`**).
- **TP2 lookup fix** `e67dd64c`: prompt-lookup copies **single-host only**.
- Same binary on both hosts: sha **`29b6fd9afa89556aca0bcff7d66398d51a57a68d16b6796e6164b8aac00e205e`**.
- Merge process documented in **`docs/MERGE-CHECKLIST.md`**.
- Users **rebooted both hosts** after Thunderbolt status-5 crash loops.
- Tag still **`tp2-rdma-merge-v0.17.0-untested`**.

## Your job (when the user allows GPU)

1. Verify `ibv_devices` / ACTIVE ports on both hosts; sha still `29b6fd9a`.
2. Confirm **no** leftover `gufo`.
3. User starts TP2 via **`tools/tp2/swap.sh` on strix-halo-1** (llama-swap)
   **or** you start **one** pair with rank0/rank1 scripts. Gufo metrics are
   on the **rank0 HTTP port**, not 8080.
4. **Short** PONG chat first. Then checklist: text generation, prompt
   **>1024 tokens**, **≥5 requests** at `--sessions 4`, one agent-style
   prompt. No multi-minute hung poll loops.
5. If **status 5** on first prefill → treat as **link/module**, not a new
   lookup bug; do not spam restarts.
6. If **size mismatch** → code regression (unexpected on this binary).
7. If green: dated tag **without** `-untested`, push **uv-genai** only.

## Repo

| | |
| --- | --- |
| Tree | `/home/ugo/projects/gufo` |
| Branch | **`main`** |
| Push | **`uv-genai` only** |
| Upstream track | local **`gufo-org`** — never push |

## Non-negotiables

- `grep -ci nix AGENTS.md` → **0**. Never run that package manager.
- One `gufo` per host. Same sha both ranks.
- Push only `uv-genai`. Never write `gufo-org`.
- After any new merge touching decode/cache/MTP: **MERGE-CHECKLIST.md**.

## Key paths

| path | purpose |
| --- | --- |
| `~/gufo-merge-handoff/HANDOFF-2026-10-19.md` | full state |
| `docs/MERGE-CHECKLIST.md` | merge + TP2 process |
| `docs/models/qwen3.8-flash-next/TP2.md` | collective rules |
| `tools/tp2/` | launchers (`swap.sh` for llama-swap) |
