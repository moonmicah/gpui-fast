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
├── nyaterm: carry the local engine patches onto the release-form branch
├── docs:    document the nyaterm branch
├── macos:   damage the native view when requesting an idle frame   (#137)
├── macos:   preserve nonblocking frame-source attachment checks    (#137)
└── fast:    wait longer each time a layer's content is still ineligible (#138)
```

The last three commits backport upstream pull requests that were open when
they were taken; see the section below for how to drop them once upstream
merges them.

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

## 2026-11-12 backport two open upstream pull requests

Both were still open upstream, and both fix behaviour NyaTerm hits, so
they ride on this branch until upstream merges them:

| PR | Upstream branch | What it fixes |
| --- | --- | --- |
| #137 | `hesong12:fix/macos-idle-redraw` | an idle macOS window stops receiving `CAMetalDisplayLink` callbacks while GPUI keeps waking its dirty window, so the frame is only painted once something else forces compositing; the shared waker now also marks the `NSView` for display through `displayLayer` (`fast/frame_demand.rs`) |
| #138 | `ilianovi3:fix/scroll-layer-ineligible-backoff` | a scroll layer whose content stays ineligible is retried every frame at 120 Hz; the wait now grows each time (`fast/layers/policy.rs`) |

They were cherry-picked onto the release-form manifests. Two conflicts
needed hand resolution: `crates/gpui_macos/src/fast/mod.rs` lists both
`cursor` and `frame_demand`, and the `window.rs` line in
`script/upstream-allowlist` now carries the frame-demand hooks alongside
the cursor exception, with the numbers from #137 (`added=48 removed=30`).

**Once upstream merges either PR**, drop the corresponding commits here
rather than keeping a duplicate: `git rebase --onto <upstream merge> <last
backport commit> nyaterm`, then re-run Verification.

## 2026-11-13 rebase onto upstream #139

Upstream main moved from `bbf6dea` to `d1d6286` (`macos: Avoid deadlocking`
`on unknown menu items (#139)`), which adds `fast/menu.rs` +
`fast/menu_tests.rs` and replaces three method bodies in
`crates/gpui_macos/src/platform.rs`. The ten NyaTerm commits were rebased
onto it. Two files conflicted twice (`fast/mod.rs` module declarations and
`script/upstream-allowlist`); both now carry the upstream `menu` entries
alongside `cursor` / `frame_demand`, and the allowlist keeps the new
`platform.rs removed=55` line from #139.

`check-upstream` still reports 36 violations, unchanged from the previous
tip, so no new upstream-hook breakage was introduced.

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
