# EigenVista coding-agent restart prompt

Snapshot: **21 October 2026, 17:31:43 AWST (UTC+08:00), Australia/Perth**.

You are continuing development of EigenVista in a fresh session. Do not assume
you have the previous chat or that its last version is the latest checkout.

## Workspace rules

Use `/Users/ugo/projects/eigenvista` as the working directory for every command,
edit, build, test, artifact, and authorized Git operation. The old checkout at
`/Users/ugo/projects/molekel`, including its `next/` directory, is read-only
reference. An inherited session cwd may still point there; correct it before
doing any work. Do not modify or build the legacy application.

Read `/Users/ugo/projects/eigenvista/AGENTS.md` first, then read in this order:

1. `/Users/ugo/projects/eigenvista/docs/session-handoff-2026-10-21_173143-AWST.md`
2. `/Users/ugo/projects/eigenvista/docs/architecture.md`
3. `/Users/ugo/projects/eigenvista/docs/development.md`
4. The latest entries in `/Users/ugo/projects/eigenvista/docs/status.md`
5. `/Users/ugo/projects/eigenvista/docs/README.md`, then the user/import/format
   guides relevant to the requested task.

The handoff is a dated snapshot, not live state. On 21 October 2026 the version
was 0.5.10, `main`/`origin/main`/tag `v0.5.10` pointed to
`69d234bbab33eb6a2a2c05c607c4d0cc2d5d3617`, and
`fixtures/large-structure/1AON.pdb` was untracked user material. Recheck all of
this. Preserve current edits and untracked files, including handoff documents.
Never read or package private `chat-*.md` or `*-chat-*.md` exports.

## Initial inspection

Create/use `/Users/ugo/projects/eigenvista/tmp` and set `TMPDIR`, `TMP`, and
`TEMP` to it before tools that create temporary files. Never use system `/tmp`.
Inspect branch/status, relevant diffs, recent Git log, remotes, manifests,
installed toolchains, and any listener on port 5178. Do not fetch/pull/reset,
switch branches, install/update dependencies, or restart a running application
merely to orient yourself. Query current GitHub issues using existing
authentication; do not print credentials. If remote access is unavailable,
state that limitation rather than assuming cached issue/remote state is current.

Recent verified builds used Rust 1.99.0 via `RUSTUP_TOOLCHAIN=stable`, Node 26,
and wasm-bindgen CLI/crate 0.2.108. Older setup docs select Rust 1.92.0;
toolchain pinning is unresolved in issue #8. Inspect the actual compiler and
its WASM target before building. Do not silently pin or upgrade it.

## Preserve the design

EigenVista is a Rust scientific core/importer/native format/CLI, with a shared
React/TypeScript/Three.js UI and Tauri desktop shell. Both UIs use the same
Rust/WASM worker for scientific calculations. No C++ implementation or copied
legacy implementation. Preserve the existing icon/logo.

Volume and sampled raycast must work directly from an orbital/density without
generating marching-cubes meshes. Keep computation cancellable, UI responsive,
and prior state intact on failure/cancellation. Preserve allocation limits,
source hashes, import validation/provenance, atomic saves, persisted meshes,
and independent scientific fixtures. Atom colours are session display state,
not chemistry. mmCIF support is a documented subset, not full crystallography.

Before a bug fix, feature, scientific/format change, or substantial refactor,
reuse an applicable GitHub issue or create one in `uvar-apps/eigenvista` using
`docs/labels.md` (area, kind, exactly one priority). Minor documentation edits
do not need a new issue. An open issue is not automatic authorization to work
on it. Do not repeat already implemented changes because a tracking issue or
older paragraph is stale.

## Execution and completion

Follow the user's current task. Keep changes scoped; verify the actual affected
workflow. Use the documented Rust, Clippy, production build, Chromium/WebKit,
and packaging checks as appropriate. Native dialogs require separate native
verification; browser tests do not cover them. Update documentation and report
tests actually run, dated prior evidence, and remaining gaps separately.

A source version bump or web build does not rebuild an existing native app or
redistributable. When those artifacts are requested, run the documented
`npm --prefix app run package:macos`, verify the resulting bundle version,
source identity and checksum, and provide exact paths. The packager rebuilds
the release app and converter and produces an ad-hoc-signed tester ZIP, not a
notarized/public release. Do not edit source during packaging or quit the user's
running app; explain that an old process must be reopened to use a new binary.

Do not create a branch/worktree, bump versions, commit, tag, push, upload,
change issue state, or start unrelated backlog implementation unless requested
by the current task. Prior chat authorizations do not carry forward.

Your first response after inspection should briefly confirm the active path,
current revision/version, preserved local changes, relevant issue and artifact
state, and the next step for the supplied task. If no implementation task is
supplied with this prompt, summarize readiness and ask what to work on next;
do not choose a feature or begin fixing the backlog yourself.
