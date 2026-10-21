You are resuming development on imagegen, an OpenAI-compatible image generation gateway backed by stable-diffusion.cpp.

## Quick start

```bash
cd /home/ugo/software/imagegen
uv sync                                    # install deps
uv run imagegen serve --config config/service.toml   # start gateway
uv run pytest                              # 100 tests, no GPU needed
uv run imagegen doctor                     # validate
scripts/fetch_models.sh                    # download all weights (~31.7 GB)
scripts/build_engine.sh cuda               # rebuild engine (also: vulkan, cpu)
docker build -t imagegen:latest .          # rebuild Docker image
```

## Current state

- **Version:** 0.1.6 · **Engine pin:** `0518b34` (`master-982-0518b34`)
- **5 served models:** `qwen-image-2.1`, `qwen-image-2.1-turbo`, `z-image-turbo`, `moss-image`, `flux-3-image-klein`
- **100 tests pass**, smoke tests pass on CUDA + Vulkan

## Architecture

```
client ──(bearer token)──> gateway :8080 ──(no credentials)──> sd-server :811x on 127.0.0.1
```

The gateway translates OpenAI `/v1/images/generations` and `/v1/images/edits` to native requests sent to supervised `sd-server` child processes. The backends are in a single-residency LRU pool (8 GB VRAM limit). Config is strict Pydantic with `extra="forbid"`.

## Key files to read first

| File | Why |
|---|---|
| `HANDOFF.md` | Full project state, architecture, models, perf, known traps |
| `AGENTS.md` | Agent-facing rules — load-bearing items, verification commands, pitfalls |
| `README.md` | User-facing docs — install, run, Docker, adding a model, license |
| `docs/architecture.md` | Design rationale, trust boundary |
| `src/imagegen/config.py` | Pydantic schemas — all model/service/backend settings |
| `src/imagegen/backend_process.py` | `build_server_args()` — profile → sd-server flags |
| `src/imagegen/backend_pool.py` | Pool lifecycle, capability cross-check |
| `src/imagegen/openai_payload.py` | OpenAI wire format → native params translation |
| `src/imagegen/routes_images.py` | Request handlers |
| `src/imagegen/cli.py` | CLI commands |
| `config/service.toml` | Shipped config with all 5 models |
| `config/container.toml` | Container variant (duplicated [[models]]) |
| `scripts/build_engine.sh` | Engine build, pin, verification |
| `tests/test_backend_process.py` | `KNOWN_SERVER_FLAGS` set |

## Key rules (from AGENTS.md)

1. **Never guess flags/APIs/artifacts** — verify against live `sd-server`, `--help`, Hugging Face API, or actual runs.
2. **`openai_payload.py` is the only module that knows the upstream wire format** — do not build prompt strings elsewhere.
3. **Every config table is `extra="forbid"`** — unknown keys are startup errors, never silent no-ops.
4. **`public_base_url` has no env override** — clients using `response_format=url` need it set in the config.
5. **`[backends].extra_args` is global** — never use for per-model flags (need a `ModelProfile` field + `build_server_args()`).
6. **Weight names are bare file names** — they resolve through `weight_path()` against `models_dir`, relative to the config file.
7. **Both configs duplicate `[[models]]`** — a model change belongs in `service.toml` AND `container.toml`.
8. **`sd-server` has no credentials** — it *must* stay on loopback (`127.0.0.1`), never exposed directly.
9. **Engine flags must be checked against `sd-server --help` of the pinned build** — never infer from model docs.

## Adding a model recipe

A `[[models]]` block in both TOML configs. If a new engine flag is needed:
1. Add field to `ModelProfile` in `config.py`
2. Emit it in `build_server_args()` in `backend_process.py`
3. Add flag to `KNOWN_SERVER_FLAGS` in `tests/test_backend_process.py`
4. Cover it in tests
5. Document in README (both key tables) and config banners

## Engine flag per model

| Model | Tokenizer | Vision tower | Edit preset |
|---|---|---|---|
| qwen-image-2.1 | embedded | `--llm_vision` | qwen (VLM) |
| qwen-image-2.1-turbo | embedded | `--llm_vision` | qwen (VLM) |
| z-image-turbo | embedded | none (no editing) | — |
| moss-image | **external (`tokenizer.json`)** → `--tokenizer` | none (no editing) | — |
| flux-3-image-klein | embedded | **none needed** (VAE-only edit preset) | flux2 (VLM=No) |

## Docker

Single multi-stage image carrying both CUDA and Vulkan engines. Selected at runtime by `IMAGEGEN_BACKEND=cuda|vulkan`. Weights in a named volume, never baked in. Preflight checks empty `/models` and prints the `fetch-models` command.

## Critical gotchas (learned the hard way)

- **`libegl1`** is a runtime dependency of the Vulkan ICD — it's `dlopen`ed, not a `NEEDED` entry, so `ldd` never reports it missing.
- **`libgomp1`** = OpenMP runtime, absent from the CUDA runtime base image.
- **Container's `Vulkan0` = RTX 2070** (only NVIDIA device passed in). On the host, `Vulkan0` = Intel iGPU.
- **The Intel iGPU produces noise** for every model including Qwen — pre-existing on this machine, not a regression.
- **`sd-server --version` + `--list-devices` need `libcuda.so.1`** — they are best-effort in `build_engine.sh` because CI/Docker build stages lack the driver.
- **`--tokenizer` was added for MOSS** (Qwen3.5 vocabulary not embedded in sd.cpp). Most models don't need it.
- **`supports_editing` no longer requires `vision_encoder`** — the flux2 edit preset is VLM=No.