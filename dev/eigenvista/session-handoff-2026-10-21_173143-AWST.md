# Restart handoff

Snapshot: **21 October 2026, 17:31:43 AWST (UTC+08:00), Australia/Perth**.
This is the concise starting
point for a new coding-agent session without this conversation. The companion
[restart prompt](restart-prompt-2026-10-21_173143-AWST.md) tells an agent how to establish context.
This snapshot is evidence, not a substitute for inspecting the next checkout.

## Workspace and authority

- Work only in `/Users/ugo/projects/eigenvista`, repository
  [`uvar-apps/eigenvista`](https://github.com/uvar-apps/eigenvista).
- `/Users/ugo/projects/molekel`, including `next/`, is read-only historical
  reference. The rewrite was extracted into EigenVista; do not develop, build,
  generate artifacts, or change Git state in Molekel.
- Explicitly set command working directories. An old chat can still start in
  Molekel even when EigenVista is the intended project.
- Read [root AGENTS.md](../AGENTS.md) first. Its workspace, issue-first,
  validation, privacy, and publication rules remain authoritative.
- Never use system `/tmp`. Keep all temporary/staging files in ignored
  EigenVista `tmp/`, with `TMPDIR`, `TMP`, and `TEMP` set before tools run.
- Preserve user changes, frozen fixtures, and provenance. Private `chat-*.md`
  and `*-chat-*.md` exports must not be read, committed, or packaged.
- No C++ implementation or copied legacy implementation. Scientific/native
  application code is Rust; shared interface code is TypeScript.
- Stay on the current branch. Branches, worktrees, commits, version bumps,
  tags, pushes, and release uploads require a current user request.
- Do not stop/restart a user's app or an unknown server. The icon/logo stays
  unchanged unless explicitly requested.

## Verified checkout snapshot

| Item | Observed state on 21 October |
| --- | --- |
| Branch | `main` |
| HEAD | `69d234bbab33eb6a2a2c05c607c4d0cc2d5d3617` |
| Release | `v0.5.10`, annotated tag pointing to HEAD |
| Origin | `https://github.com/uvar-apps/eigenvista.git` |
| Remote | Live `origin/main` and peeled `v0.5.10` both match HEAD |
| Manifests | Rust workspace, npm package/lock root, and Tauri are `0.5.10` |
| Pre-existing local work | Untracked `fixtures/large-structure/1AON.pdb`; leave it untouched |
| Preview server | No listener on port 5178 at inspection time; check again before starting |
| Built release app | Its bundle metadata reports `0.5.10` |

This handoff adds documentation changes after that inspection; they are not
part of the existing tag or ZIP until explicitly committed/repackaged. No
feature implementation is currently assigned by this handoff. Reorient first,
then follow the user's next task rather than autonomously picking backlog work.

## Product and implemented workflows

EigenVista is a molecular and quantum-chemistry data viewer, not an electronic
structure solver. It is a development preview, not completion of v1 or the
historical M0-M7 plan. Apple Silicon macOS is the exercised native platform;
Linux and Windows remain targets, not qualified releases.

- Open native `.eigenvista`, legacy preview `.molekel`, XYZ, PDB, mmCIF
  (`.cif`, `.mcif`, `.pdbx`), supported Molden files, and supported Gaussian
  cube scalar fields. Import loss/normalization reports must remain explicit.
- All supported imports receive coordinate-based display bonds. Explicit
  connections are preserved where supported; inferred bonds are not bond orders.
- Molden supplies basis functions, orbitals, and occupation-derived densities
  under documented conventions. ORCA's Molden export is the intended route,
  with a tested subset; there is no direct ORCA or Gaussian-log reader.
- An orbital browser lists metadata, filters/selects fields, and batches up to
  32 orbitals for persistent signed meshes. Existing unselected meshes survive.
- Selecting volume or sampled raycast for an orbital/density must work without
  marching cubes. Only explicit surface generation requests meshing. Changing
  field/resolution in a sampled mode resamples in the background; cancellation
  or failure preserves the previous field, resolution, preview, and geometry.
- Cubes retain authoritative f64 samples and affine axes, including supported
  skewed/reflected grids. Display resampling must not alter source values.
- Meshes and generation/material metadata are saved with scientific inputs in
  native documents. Reopening saved meshes does not require recalculation.
- Instanced atoms/bonds support large structures, picking, and three molecular
  representations. Zoom uses wheel, middle drag, and toolbar buttons.
- Element symbols/radii cover Z 1-118. A validated JSON palette can override
  atom colours for the session, with reset to bundled defaults. Palette changes
  re-tint instances; they do not alter chemistry or persist in native documents.
- Custom vertex/fragment surface shaders, live uniforms, and parameter files
  exist. Native file filters were fixed in 0.5.10; other shader issues remain.
- The shared Rust importer supports Open, GUI batch conversion, and the
  `eigenvista-convert` CLI. No Python or vendor program is required at runtime.

## Architecture and invariants

Read [architecture](architecture.md) and [native format](preview-format.md)
before shared-contract changes. Key ownership:

| Path | Responsibility |
| --- | --- |
| `crates/eigenvista-core/src/model.rs` | Validated scientific model and allocation limits |
| `crates/eigenvista-core/src/compute.rs` | AO/field evaluation, sampling, gradients, meshing |
| `crates/eigenvista-core/src/bonds.rs`, `src/import/` | k-d-tree bond search and structure readers |
| `crates/eigenvista-import/src/lib.rs` | Shared format dispatch, Molden, provenance/reports |
| `crates/eigenvista-format/` | Native ZIP/binary arrays, hashes, validation, atomic saves |
| `crates/eigenvista-convert/` | CLI conversion/check/batch workflows |
| `crates/eigenvista-wasm/src/lib.rs` | WASM bridge and Rust-owned sampled-grid cache |
| `app/src/science.ts`, `science.worker.ts` | Request IDs, worker jobs, cache, cancellation |
| `app/src/App.tsx`, `OrbitalBrowser.tsx` | Document/preview state and orbital selection |
| `app/src/Viewport.tsx`, `volume.ts` | Three.js/WebGL2 scene, camera, sampled rendering |
| `app/src/elements.ts`, `atomColors.ts`, `atom-colors.json` | Display radii/symbols and session palette |
| `app/src/files.ts`, `ShaderPanel.tsx` | Browser/native file adapters and shader controls |
| `app/src-tauri/src/main.rs` | Tauri shell, native save/version commands |
| `tools/run.mjs`, `tools/package-macos.mjs` | Builds/tests and verified tester ZIP creation |

Both browser and desktop scientific jobs currently run in a **Rust/WASM Web
Worker**, not a native desktop thread pool. Rust also builds natively for tests,
the converter, and native save validation. No server/database is required.

Preserve these contracts:

- Scientific values and native arrays are f64; scene coordinates are bohr,
  picked positions display angstroms, supplied orbital energies are hartree.
  GPU values/geometry and mesher interpolation have f32 limitations.
- Real explicit Gaussian-polynomial functions through G, real orbitals and
  supplied density matrices are supported. Do not invent absent quantum data.
- Grid layout is x-fastest; axes are step vectors, not full extents. Preserve
  scientific hashes, IDs, validation, checksums, and atomic-save semantics.
- Transient analytic grids allow up to 256 cubed; authoritative imported/native
  grids retain 128 cubed total samples. The UI defaults to 40 per axis.
  Keep the two-million-vertex, 128-surface, 256-AO, and native 128 MiB bounds.
  Recoverable errors must preserve state; browser/GPU/system OOM is not always
  recoverable. No blanket timing or memory guarantee follows from small tests.
- Meshing uses classic `mcubes`, not topology-certified MC33. Sampled raycast
  and volume rendering remain experimental, with mixed-scene transparency and
  sampling limitations. Direct analytic shader raycasting is not implemented.
- mmCIF is a bounded text subset: first model and PDB-like alternate selection,
  coordinate bonds, recorded cell/symmetry metadata. No `struct_conn`, symmetry
  expansion, periodic boundaries, BinaryCIF, or mmCIF export yet. See
  [structure imports](structure-imports.md) for malformed-input behavior and
  the rationale for avoiding the upstream parser's panic paths.

## Re-establish the environment

Run these inspection commands before installing/updating dependencies or editing:

```sh
cd /Users/ugo/projects/eigenvista
mkdir -p tmp
export TMPDIR="$PWD/tmp" TMP="$PWD/tmp" TEMP="$PWD/tmp"
pwd
git status --short --branch
git diff --stat
git diff --cached --stat
git log -8 --oneline --decorate
git remote -v
rustup show active-toolchain
rustc --version
node --version
wasm-bindgen --version
lsof -nP -iTCP:5178 -sTCP:LISTEN
```

No listener makes `lsof` exit 1; it is not an application failure. If a listener
exists, inspect its process/cwd before reusing it. Do not kill it automatically.
Review relevant diffs without opening private chat exports.

Observed toolchain: `stable-aarch64-apple-darwin`, rustc **1.99.0**, Node
**26.0.0**, wasm-bindgen CLI **0.2.108**. The CLI must match the pinned Rust
crate. Cargo/npm lockfiles pin dependencies, not the Rust compiler. Setup docs
still select 1.92.0 via an environment variable; issue #8 tracks the unresolved
toolchain policy. Do not silently add a compiler pin or update dependencies.

For the same currently installed stable toolchain used in recent verification:

```sh
export RUSTUP_TOOLCHAIN=stable
rustc --version
rustup target list --installed --toolchain stable
```

If missing, install `wasm32-unknown-unknown` for that exact chosen toolchain;
an installed target for 1.92.0 does not cover stable. `stable` can change over
time, so record the actual compiler version. A fresh setup also needs Xcode
command-line tools, `npm --prefix app ci`, and matching wasm-bindgen CLI.
Follow [development](development.md) for prerequisites; Python/PySCF is only
for intentionally regenerating independent reference data, not routine builds.

## Run, verify, and package

From EigenVista with the temp environment above:

| Need | Command |
| --- | --- |
| Web development | `npm --prefix app run dev` |
| Desktop development | `npm --prefix app run desktop` |
| Production WASM/frontend only | `npm --prefix app run build` |
| Debug desktop bundle | `npm --prefix app run desktop:build` |
| Optimized release app + converter + tester ZIP | `npm --prefix app run package:macos` |
| CLI | `cargo run --locked -p eigenvista-convert -- --help` |

Web/desktop development both use strict port 5178. Choose one, inspect ownership,
and follow the coordinated port configuration instructions if occupied.
**Building the web frontend or bumping a version does not rebuild a native app
or ZIP.** This caused the earlier disabled volume/raycast report: source was
0.5.4 while the launched bundle was 0.5.3. Rebuild the requested artifact and
check its own metadata. An already-running process still needs quit/reopen by
the user; do not terminate it yourself without authorization.

Runtime verification baseline (scale to the changed behavior):

```sh
cargo fmt --all --check
cargo test --locked --workspace
cargo clippy --locked --workspace --all-targets -- -D warnings
npm --prefix app run build
npm --prefix app run test:e2e
npm --prefix app run test:packaging
```

Install Playwright Chromium/WebKit if absent per the development guide. Default
`cargo test` excludes the desktop shell; `--workspace` includes it. Build WASM
before browser tests when reusing an existing server. Browser tests cover real
inputs/downloads, independent science, cancellation, canvas pixels, camera and
mobile layout; inspect screenshots as well. They do **not** exercise native
Tauri file dialogs. Test native changes separately and state coverage limits.

Useful inputs: `fixtures/legacy/data/molden.input` (17 atoms, 125 AOs, 118
orbitals and an occupation-derived density), `fixtures/molden/`,
`fixtures/pyscf/`, `fixtures/cube/signed-affine.cube`, legacy cube/PDB fixtures,
`fixtures/cif/1BH3.cif`, `fixtures/cif/1AON.cif`, and
`fixtures/large-structure/4PJ1.pdb`. Preserve originals. Generated example native
files live in ignored `artifacts/references/`. Do not mistake the untracked
`fixtures/large-structure/1AON.pdb` for an approved regression fixture.

## Artifacts and evidence

Latest local package observed:

```text
/Users/ugo/projects/eigenvista/artifacts/distributions/EigenVista-0.5.10-apple-silicon-20261012T140526Z-69d234b.zip
```

Its same-stem `.json` and `.zip.sha256` companions are present. The checksum
was rechecked on 21 October:
`60914af3142729059a8ecb6cd72d8baeb391ac026cfd3bb7f9d2697cc345831d`.
Metadata records clean source at `69d234b`, Rust 1.99.0, arm64/macOS 27 build
host, deployment target macOS 13, ad-hoc signing, and **not notarized**.
These local ignored artifacts are not guaranteed to exist in a fresh clone.

Rebuilt local app:
`target/aarch64-apple-darwin/release/bundle/macos/EigenVista.app`.
Share the **whole tester ZIP**, not just that raw target bundle: the staged
package includes notices, licenses, converter, examples, and exact source.
The repeatable packaging command verifies builds, imports/roundtrips,
signatures, extracted files/modes, and source stability. It never uploads,
notarizes, commits, or restarts the user's app. Do not edit source while it runs.
See [tester packaging](tester-packaging.md) for all boundaries.

Evidence is dated, not a new test claim:

- 12 October 0.5.10 [release report on #23](https://github.com/uvar-apps/eigenvista/issues/23#issuecomment-6125936422)
  records 141 default-member Rust tests, separate desktop-shell pass, 150
  Chromium/WebKit cases, strict Clippy, and verified packaging. Package metadata
  records build/signature/import/extraction checks; packaging suite has 13 tests.
- A later [user confirmation on #23](https://github.com/uvar-apps/eigenvista/issues/23#issuecomment-6126068761)
  verifies shader/params native dialogs in shipped 0.5.10. Issue #23 is closed.
  That does not prove every native workflow, including palette dialogs, works.
- This 21 October handoff pass inspected Git, manifests, issue states, tools,
  bundle metadata, and the package checksum. It did **not** rerun runtime tests
  or build an app. Clean-machine/Gatekeeper installation, the full macOS range,
  and Linux/Windows remain unqualified. No hosted CI is configured.

## Open work, not an instruction to implement it

GitHub open-issue snapshot on 21 October; re-query before planning:

| Priority | Issues |
| --- | --- |
| P1 | [#3](https://github.com/uvar-apps/eigenvista/issues/3) global-max-radius bond search; [#6](https://github.com/uvar-apps/eigenvista/issues/6) cancellation rejects every pending worker request; [#9](https://github.com/uvar-apps/eigenvista/issues/9) shader parameter type mismatch crashes |
| P2 | [#2](https://github.com/uvar-apps/eigenvista/issues/2) fixed 160-step volume march; [#5](https://github.com/uvar-apps/eigenvista/issues/5) full-document JSON cache keys; [#8](https://github.com/uvar-apps/eigenvista/issues/8) compiler pin policy; [#10](https://github.com/uvar-apps/eigenvista/issues/10) native parameter save path; [#11](https://github.com/uvar-apps/eigenvista/issues/11) wasm32 column-index wrapping; [#12](https://github.com/uvar-apps/eigenvista/issues/12) custom shader opacity/blending; [#16](https://github.com/uvar-apps/eigenvista/issues/16) colour-table tracking |
| P3 | [#4](https://github.com/uvar-apps/eigenvista/issues/4) native platform label; [#7](https://github.com/uvar-apps/eigenvista/issues/7) retained drawing buffer; [#19](https://github.com/uvar-apps/eigenvista/issues/19) mmCIF declared connections; [#20](https://github.com/uvar-apps/eigenvista/issues/20) crystallographic symmetry/periodicity; [#21](https://github.com/uvar-apps/eigenvista/issues/21) BinaryCIF; [#22](https://github.com/uvar-apps/eigenvista/issues/22) mmCIF export |

Issue #16 is still open even though 0.5.7/0.5.8 contain the colour fixes.
Inspect its history/code before treating it as unimplemented or closing it.
No issues were modified during this handoff pass.

For bug fixes, features, scientific/file-format changes, or substantial
refactors, reuse an applicable issue or open one **before implementation**.
Use the [label taxonomy](labels.md): area(s), kind(s), exactly one priority.
Minor documentation-only work such as this handoff does not require a new issue.
Update behavior docs and record actual checks/gaps; do not claim all planned
M0-M7 work complete. The [documentation map](README.md) links the frozen
original brief/research and the current user, format, scientific, and developer
guides. [Historical handoff](handoff.md) retains the longer migration/release story.
