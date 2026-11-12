# NyaTerm fork notes

`nyaterm` carries the engine patches NyaTerm needs on top of
`longbridge/gpui-fast`. NyaTerm consumes this branch as a git dependency
pinned by revision; there is no vendored copy in the application.

## Layout

```
main      follows longbridge/gpui-fast
nyaterm
├── release: use the published registry manifests for the release crates
├── build:   wire the release crates by path inside this workspace
├── fast:    answer CursorStyle::Hidden with a transparent native cursor
├── docs:    register the CursorStyle::Hidden exception with its PR number
└── nyaterm: carry the local engine patches onto the release-form branch
```

## 2026-11-12 move NyaTerm onto a fork branch

Upstream does not accept new public APIs, so the patches that NyaTerm
needs can never land in `longbridge/gpui-fast`. Hosting them here keeps
the engine upgradeable without maintaining a vendored source dump.

The nine release crates use the same manifests that ship to crates.io
(`gpui` -> `gpui-fast`, `gpui_macos` -> `gpui-fast-macos`, ...), with the
library names unchanged, so a consumer resolves this branch exactly like
the published packages. Snapshots of the non-release crates still come
from the registry (`gpui-pre-*` 0.3.12), as they do for a published
release.

Patch groups, each taken from NyaTerm's vendored 0.1.14 tree and
three-way merged with current upstream sources:

| Group | What it adds |
| --- | --- |
| native overlay | `elements/native_overlay.rs` and the macOS/Windows window-hosted overlays that draw sub-windows above the app surface |
| external drag payload | `VirtualFiles` / `VirtualFileTree` / `PromisedFiles` in `ExternalDragPayload`, `file_promise.rs`, `virtual_file.rs`, `NSFilePromiseProvider` drag-out on macOS, OLE virtual-file streams on Windows |
| cursor hidden | `CursorStyle::Hidden` answered by a transparent native cursor (macOS, Windows) so remote-desktop frames can hide the local pointer |
| pointer cancellation | `PlatformInput::PointerCancelled`, delivered when a window loses the pointer capture mid-drag |
| accessibility checkpoint | accessibility state rolled back when a prepaint retry restarts a frame, plus the `has_internal_drag` and `activate_accessibility` helpers the app and its tests use |

The Windows dynamic-texture backend is **not** carried here: upstream
#131 already ships it.

## Verification

- `cargo check -p gpui-fast -p gpui-fast-macos` (macOS host) passes.
- `cargo check -p gpui-fast -p gpui-fast-windows --target
  x86_64-pc-windows-msvc --features gpui-fast/windows-manifest-provided-by-host`
  passes; the feature skips the Windows resource compiler, which a macOS
  host does not have.
- Linux is **not** verified here: `cargo check --target
  x86_64-unknown-linux-gnu` stops in a C dependency because the host has
  no `x86_64-linux-gnu-gcc`.
- `gpui-fast-apple` builds with `runtime_shaders`; the macOS host has no
  offline Metal compiler.

## Upgrading

1. `git fetch origin && git merge vX.Y.Z` on `nyaterm`; conflicts land
   only in the patch groups above.
2. Re-run the three checks in Verification.
3. Bump `rev` in NyaTerm's `[patch.crates-io]` entries to the new tip.
