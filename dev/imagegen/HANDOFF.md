# imagegen — Handoff Document
**Version:** 0.1.6 · **Engine pin:** `0518b34` · **Date:** 2026-10-21
**Unreleased:** `layout` `/v1/models` extension on `master` after tag `v0.1.6`
(commit `2e71ac8`; no version bump yet — the next release is expected to be 0.1.7)

---

## Overview

An OpenAI-compatible HTTP gateway in front of `sd-server` (stable-diffusion.cpp) serving five image-generation models. The gateway performs no inference — all sampling happens in supervised child processes.

**Repository:** `https://github.com/uv-genai/imagegen`  
**Working tree:** `/home/ugo/software/imagegen`  
**Language:** Python 3.12+ (FastAPI), engine in C++ (vendored in `third_party/`)

---

## Quick commands

```bash
uv sync                          # install deps
uv run pytest                    # 110 tests, no GPU needed
uv run imagegen serve            # start the gateway (loopback by default)
uv run imagegen doctor           # validate config, binaries, weights
uv run imagegen models           # list configured models with sampling
uv run imagegen plan --model x   # show exact sd-server cmd line for model x
scripts/fetch_models.sh          # download all weights (~31.7 GB, resumable)
scripts/build_engine.sh cuda     # build CUDA engine (also: vulkan, cpu)
docker build -t imagegen:latest .  # rebuild Docker image
```

---

## Architecture

```
client ──(bearer token)──> gateway :8080 ──(no credentials)──> sd-server :811x on 127.0.0.1
```

- **Gateway** (`/v1/images/generations`, `/v1/images/edits`, `/v1/models`) — translates OpenAI shapes to `sd-server` native params. The only authenticating layer.
- **Backend pool** (`backend_pool.py`) — LRU-evicting pool of supervised `sd-server` processes. `max_resident_backends=1` (9.7 GB weights exceed 8 GB VRAM). On-demand lifecycle; idle unload at 300 s.
- **Config** (`config.py`) — strict Pydantic schemas with `extra="forbid"`. All 41 keys documented in README.

## Models served

| Model id | Architecture | Steps | CFG | Edit? | License |
|---|---|---|---|---|---|
| `qwen-image-2.1` | 7B DiT | 40 | 6.0 | yes (VLM) | Non-commercial |
| `qwen-image-2.1-turbo` | 7B DiT distilled | 8 | 1.0 | yes (VLM) | Non-commercial |
| `z-image-turbo` | 6B DiT distilled | 8 | 1.0 | no | **Apache-2.0** |
| `moss-image` | 5B DiT (SFT) | 20 | 3.0 | no | **Apache-2.0** |
| `flux-3-image-klein` | 4B DiT distilled | 16 | 1.0 | **yes (VAE-only, no VLM)** | FLUX Kommunity |

### Key details per model

- **MOSS:** uses Qwen3.5-9B text encoder (external `tokenizer.json`), FLUX.2 VAE. `dimension_multiple=16`. Time shift 5.0. Weights from `realrebelai/MOSS-Image_GGUFs`.
- **flux3:** uses Qwen3-VL-4B (embedded tokenizer), FLUX.2 VAE. `dimension_multiple=16` (verified empirically at 1008×1008). Time shift 6.93. Edits use the `flux2` preset (VLM=No) — `supports_editing=true` with `vision_encoder=None`, which the schema now allows.
- **Z-Image:** uses Qwen3-4B-Instruct-2507 (no mmproj), FLUX.1-schnell autoencoder (`ae.safetensors`). `dimension_multiple=16`. Apache-2.0.
- **Qwen models:** share Qwen3-VL-8B encoder + mmproj. `dimension_multiple=32`. Turbo has a fixed 9-element sigma schedule.

---

## Discovery: the `layout` extension (unreleased, after v0.1.6)

`GET /v1/models` is the only discovery surface. Each entry MAY carry
`"layout": "<dialect>"`; absent means "not advertised". **Never `null`** — clients
distinguish "this model has no layout support" from "this engine predates the
extension", and `null` erases that difference.

`"moss"` means the checkpoint parses a MOSS layout JSON document as the ordinary
`prompt` string — `summary.caption`, `items` with `label`/`caption`/`text`/`bbox`,
`bbox_format: "yxyx"`, `bbox_coord_space: "abs_centralized"` (canvas-centre origin,
so 1024×1024 spans −512…+512). The gateway parses nothing; it relays the string.

- Set per profile via `layout = "moss"`; shipped on `moss-image` only.
- Layout support lives in the **weights** (both MOSS Base and SFT document it), so a
  non-MOSS model must not claim the dialect.
- Verified end-to-end: the upstream USERGUIDE example layout rendered to its exact
  bboxes through the real MOSS checkpoint on CUDA (180 s at 1024²).
- Layout is *experimental* upstream, and bboxes are absolute pixels for one canvas,
  so a client must generate at the size it computed coordinates for.

---

## Perf characteristics (RTX 2070 8 GB, CPU offload, 512×512 unless noted)

| Model | Time | Notes |
|---|---|---|
| Qwen Turbo | ~16 s | letter-exact, text-heavy default |
| Qwen base | ~88-98 s | 40 steps, higher quality |
| Z-Image 1024² | ~34 s | fast sketcher, garbled captions (3/3 seeds) |
| flux3 1024² | ~58-71 s | fastest full-res, letter-exact, layout can be sparse |
| MOSS 1024² | ~202 s | best quality (correct icons), 20 steps + 9B encoder |
| flux3 edit 512² | ~32 s | VLM=No edit preset |
| Qwen edit 512² | ~25 s | VLM edit preset |
| Model swap | ~2 s | |
| Vulkan1 (RTX 2070) | ~2-3× CUDA | Vulkan0 = Intel iGPU → noise for all models |

---

## Perf characteristics (Docker container, RTX 2070 + CUDA)

| Configuration | Time |
|---|---|
| CUDA MOSS 512² cold | 54 s |
| Vulkan flux3 512² cold | 31 s |
| Compose flux3 512² (page-cached) | 15 s |

In-container, Vulkan0 = RTX 2070 (only NVIDIA device passed in; no iGPU).

---

## Key engine changes this pin

Pin moved from `f89d9b1` → `0518b34` (31 commits, ~Oct 10–21). Relevant upstream changes:
- `68285df` — MOSS-Image support
- `8cd731e` — FLUX.3 Image Klein
- `95b3090` — MOSS split QKV + batch host loading
- `9b7da27` — fix OpenAI edit routing by reference conditioning support

---

## Notable config points

- `extra="forbid"` on every table: unknown keys are startup errors, not silent no-ops.
- `public_base_url` has **no environment override** — must be edited in the file or mounted config.
- `[backends].extra_args` is global — never use for per-model flags. Per-model flags need a `ModelProfile` field and `build_server_args()` emission.
- `supports_editing` and `vision_encoder` are independent: flux3 edits with no tower.
- Weight names resolve through `BackendSettings.weight_path` against `models_dir`, relative to the config file.

---

## Schema reference (all 41 keys)

### `[service]` — 8 keys
`host`, `port`, `api_keys`, `public_base_url`, `request_timeout_seconds`, `max_waiting_requests`, `blob_ttl_seconds`, `blob_capacity`

### `[backends]` — 12 keys
`sd_server_binary` (req), `models_dir` (req), `lifecycle` (eager|on_demand), `max_resident_backends`, `idle_timeout_seconds`, `start_timeout_seconds`, `stop_timeout_seconds`, `offload_to_cpu`, `diffusion_flash_attention`, `max_vram_gib`, `log_level`, `extra_args`

### `[[models]]` — 15 keys
`id` (req), `diffusion_model` (req), `vae` (req), `text_encoder` (req), `port` (req), `default_size` (req), `max_pixels` (req), `sampling` (req), `description`, `vision_encoder`, `tokenizer` (0.1.6), `supports_editing`, `layout` (unreleased), `dimension_multiple` (default 32), `max_images_per_request`

### `[models.sampling]` — 7 keys
`steps` (req), `cfg` (req), `sample_method` (req), `scheduler`, `flow_shift`, `custom_sigmas`, `negative_prompt`

---

## Environment variables

| Variable | Source | Overrides |
|---|---|---|
| `IMAGEGEN_API_KEYS` | `cli.py` | `service.api_keys` |
| `IMAGEGEN_HOST` | `cli.py` | `service.host` |
| `IMAGEGEN_PORT` | `cli.py` | `service.port` |
| `IMAGEGEN_BACKEND` | `entrypoint.sh` | cuda|vulkan (container only) |
| `IMAGEGEN_CONFIG` | `entrypoint.sh` | which config file to load (container only) |
| `IMAGEGEN_MODELS_DIR` | `docker-compose.yml` | override named volume with host path (container only) |
| `CUDA_ARCHITECTURES` | `build_engine.sh` | nvcc target arch (build only) |

No environment override exists for `public_base_url`.

---

## Docker implementation

- `Dockerfile` — multi-stage: CUDA engine on `nvidia/cuda:13.3.1-devel`, Vulkan engine on `ubuntu:24.04` (needs `glslc` + `spirv-headers`), runtime on `nvidia/cuda:13.3.1-runtime` with `libvulkan1 libgomp1 libegl1 libglx0 libx11-6 libxext6`.
- `docker/entrypoint.sh` — sets `engine/current` symlink based on `IMAGEGEN_BACKEND`, preflight checks for weights in `/models`, dispatches `serve`/`fetch-models`/`doctor`/`models`/`plan`.
- `config/container.toml` — duplicate of `service.toml` with container paths and `host = "0.0.0.0"`. Both must stay in sync for `[[models]]`.
- `docker-compose.yml` — `IMAGEGEN_API_KEYS` required; `IMAGEGEN_MODELS_DIR` for host bind-mount; `NVIDIA_VISIBLE_DEVICES=all` ensures Vulkan ICD is mounted.
- Weights never baked in (~31.7 GB). Named volume + `fetch-models` one-shot.

---

## Files to read before modifying

| File | Purpose |
|---|---|
| `src/imagegen/config.py` | Pydantic schemas — all 5 model settings classes |
| `src/imagegen/backend_process.py` | `build_server_args()` — translates profile to sd-server flags |
| `src/imagegen/backend_pool.py` | `capability_mismatches()`, pool lifecycle |
| `src/imagegen/openai_payload.py` | Raw OpenAI wire format — extension block, field merge |
| `src/imagegen/routes_images.py` | Request handlers, validation |
| `src/imagegen/cli.py` | `imagegen doctor/models/plan/serve` |
| `tests/test_backend_process.py` | Flag tests, `KNOWN_SERVER_FLAGS` set |
| `tests/test_config.py` | Schema tests |

---

## Testing discipline

- No GPU needed for 100 pytest tests (stub `sd-server`). The two weight-dependent tests self-skip.
- Real smoke test: `scripts/smoke_test.py --base-url ... --api-key ... --model ...`.
- Pin bumps require: rebuild → re-check `sd-server --help` → add new flags to `KNOWN_SERVER_FLAGS` → re-run smoke tests on at least one existing model (regression) → re-check capabilities payload.
- Engine flags must be verified against `sd-server --help` of the **pinned build**, never inferred.
- Byte sizes in `fetch_models.sh` come from Hugging Face API (`/api/models/<repo>?blobs=true`), never estimated.

---

## Known issues / traps

1. **iGPU (Intel UHD 630, Vulkan0)** produces noise for every model including Qwen — treat as unverified, not fixed.
2. **GPU throttles at 82 °C** on this machine, causing ±20% variance in CUDA times.
3. **`public_base_url` has no env override** — clients using `response_format=url` get broken links unless the config is edited.
4. **`[backends].extra_args` is global** — a flag set there is handed to *every* model's `sd-server`.
5. **`supports_editing` + no `vision_encoder`** is now legal (flux3) but the edit endpoint still validates `supports_editing` — a model with both `false` and no tower gives a clear error.
6. **Container config duplicates `[[models]]`** from `service.toml` — model changes belong in both.
7. **`--tokenizer` is only for external vocabularies** (MOSS/Qwen3.5). Most models embed their tokenizer in the GGUF.
8. **Vulkan0 inside Docker** is the RTX 2070 (only NVIDIA device passed in), but on the host Vulkan0 is the Intel iGPU. Device order is loader-dependent, not speed-dependent.
9. **The `sd-server --version` check in `build_engine.sh` is best-effort** — it fails gracefully on driverless hosts (CI, Docker build stage) because it needs `libcuda.so.1`.

---

## What could come next (no priority implied)

- GitHub release for `v0.1.6`
- Add Ming-Image (needs `--tokenizer`, which now exists as a field; also needs a Ling-mini-2.0 encoder)
- Add Krea2, LLaDA-Image, or other architectures that landed upstream
- Multi-GPU support (multiple gateway instances behind a load balancer; `--backend` device pinning)
- Job polling API (re-expose `/sdcpp/v1/img_gen`)
- `response_format=url` fix for bare metal (bind-mount config override or env override for `public_base_url`)
- User accounts / per-key quotas (belongs at a reverse proxy)
- Docker Hub push (`ugovaretto/imagegen`)