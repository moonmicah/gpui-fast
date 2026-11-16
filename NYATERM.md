# NyaTerm fork notes

`nyaterm` carries the engine patches NyaTerm needs on top of
`longbridge/gpui-fast`. NyaTerm consumes this branch as a git dependency
pinned by revision; there is no vendored copy in the application.

## Layout

```
main      follows longbridge/gpui-fast
nyaterm
├── release: use the published registry manifests for the release crates
├── fast:    answer CursorStyle::Hidden with a transparent native cursor
├── nyaterm: carry the local engine patches onto the release-form branch
├── macos:   request native damage when waking an idle Metal window (#137)
└── docs:    document the nyaterm branch and the rebases it took
```

Related work shares a commit: the `release:` commit also wires the crates by
path inside this workspace, `fast:` also registers the CursorStyle::Hidden
exception, and `macos:` carries both halves of upstream #137, which was still
open when it was taken; see the section below for how to drop it once upstream
merges it. #138 was carried the same way but upstream shipped it in v0.1.15,
so its commit was dropped during the 2026-11-13 rebase.

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

**#138 landed upstream as `e1ed545` (tag `v0.1.15`)** with a patch-id
identical to the cherry-picked commit, so the 2026-11-13 rebase dropped it
automatically. **Once upstream merges #137**, drop that commit here
rather than keeping a duplicate: `git rebase --onto <upstream merge> <last
backport commit> nyaterm`, then re-run Verification.

## 2026-11-13 rebase onto upstream #139

Upstream main moved from `bbf6dea` to `d1d6286` (`macos: Avoid deadlocking`
`on unknown menu items (#139)`), which adds `fast/menu.rs` +
`fast/menu_tests.rs` and replaces three method bodies in
`crates/gpui_macos/src/platform.rs`. The NyaTerm commits were rebased
onto it. Two files conflicted twice (`fast/mod.rs` module declarations and
`script/upstream-allowlist`); both now carry the upstream `menu` entries
alongside `cursor` / `frame_demand`, and the allowlist keeps the new
`platform.rs removed=55` line from #139.

`check-upstream` still reports 36 violations, unchanged from the previous
tip, so no new upstream-hook breakage was introduced.

## 2026-11-13 rebase onto upstream v0.1.15 + #140

Upstream main moved from `d1d6286` to `80be501` with two commits:

- `e1ed545` (tag `v0.1.15`) is #138, the scroll-layer backoff previously
  backported here. Its patch-id matches the local cherry-pick exactly, so
  `git rebase --onto origin/main d1d6286 nyaterm` dropped the duplicate.
- `80be501` is #140, a Windows-only change (`fast/backdrop.rs` plus two
  edits in `gpui_windows`) making Blurred windows readable with the Windows
  11 acrylic backdrop. It touches no file in the local patch groups.

The remaining NyaTerm commits replayed without a conflict: #139 was
already in the new base, and the `fast/mod.rs` / `upstream-allowlist`
resolutions from the previous rebase applied cleanly. `fast/mod.rs` now
lists `composition`, `cursor`, `frame_demand` and `menu` together.

## 2026-11-16 rebase onto upstream #143 + #144

Upstream main moved from `80be501` to `cd2e68f` with two commits:

- `719b645` is #143: the web backend falls back to the CSS size when
  `devicePixelContentBoxSize` ignores `devicePixelRatio`, in the new
  `gpui_web/src/fast/canvas_size.rs`. No local patch group is touched.
- `cd2e68f` is #144: `fast/pointer.rs` remembers a `MouseExited` position and
  hit testing ignores that last position until the next pointer event, so a
  hitbox flush with the window edge stops reporting hover. It replaces
  `rendered_frame.hit_test(mouse_position)` with `fast::pointer::hit_test` in
  both `prepaint` and `dispatch_mouse_event`.

One conflict, in `dispatch_mouse_event`: the native-overlay outside-down block
sat exactly on the line #144 rewrote. The resolution keeps upstream's
`dependencies::read_pointer` and `pointer::hit_test` calls and leaves the
overlay block between them, so the overlay still sees the press before the hit
test is taken.

The NyaTerm commits replayed otherwise clean; `check-upstream` still
reports 36 violations, unchanged.

## 2026-11-16 rebase onto upstream #146

Upstream main moved from `cd2e68f` to `31a0d8f` (`window: Defer window
callbacks that arrive while the app is borrowed (#146)`), which fixes the
`RefCell already borrowed` errors and the `on_display_changed` panic a
re-entered Windows window procedure caused. It adds
`crates/gpui/src/fast/misc/reentrant_window.rs`, registers it with one line
in `crates/gpui/src/window.rs`, makes `Window::display_changed` `pub(crate)`
for it, and raises that file's allowlist entry from `added=108` to
`added=110`.

The five NyaTerm commits replayed without a conflict: #146's hook sits just
above `on_move_tab_to_new_window`, away from the hooks the `nyaterm:` commit
adds, and its allowlist entry applied as-is.

`check-upstream` still reports 36 violations, unchanged.

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
