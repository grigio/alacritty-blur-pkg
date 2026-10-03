# Upstream report: enabling `ext-background-effect-v1` blur in Alacritty

Prepared from the `alacritty-blur` 0.17.0 packaging (local backport of the protocol
into winit 0.30.13). All facts verified against upstream sources on 2026-10-03.

**TL;DR**

| where | status | what is still needed |
|---|---|---|
| winit | ✅ already merged, shipped in `0.31.0-beta.3` | release `0.31.0` final — and, ideally, backport to the `0.30.x` line |
| alacritty | ✅ feature is fully plumbed, no protocol code to write | bump `winit` to `0.31` (PR #8750, rebase to beta.3) + docs update |
| wayland-protocols | ✅ nothing | — (`staging/ext-background-effect` already in `0.32.8+`) |

---

## 1. Report for **rust-windowing/winit**

### Already done (no code change required)

The protocol support exists and is released:

- Commit `c4afadb` — *"winit-wayland: use `ext-background-effect` if available"*
  (2026-04-04) added:
  - `winit-wayland/src/types/ext_background_effect.rs` — manager/surface wrapper,
    binds `ext_background_effect_manager_v1` `1..=1`
  - `winit-wayland/src/types/bgr_effects.rs` — `BgrEffectManager::{Ext, KWin}`
    enum: **prefers `ext-background-effect-v1`, falls back to `org_kde_kwin_blur`**
- It is contained in the **`v0.31.0-beta.3`** tag and was published to crates.io
  as `winit 0.31.0-beta.3` on **2026-09-04** (verified: the two files are present
  at that tag and in that release).
- Changelog `winit/src/changelog/v0.31.md`: *"On Wayland, added
  ext-background-effect-v1 support."*
- Behaviour is correct and complete for a terminal use case:
  - `winit-wayland/src/window/state.rs:678` `set_blur()` creates the effect and
    sets the region to `0,0,i32::MAX,i32::MAX` (spec: clipped by the compositor
    to the surface size, so it stays valid across resizes).
  - `winit-wayland/src/window/state.rs:~499` re-applies the region on every
    configure/resize.
  - `winit-wayland/src/window/common.rs:160-165` calls `request_redraw()`
    whenever the change needs a `wl_surface.commit` — required, because
    `set_blur_region` is double-buffered state.
  - `SurfaceBlurEffect::drop()` destroys the `ext_background_effect_surface_v1`
    object (the spec removes the effect on the next commit).

### What is still needed

1. **Publish `0.31.0` final.**
   `0.31.0-beta.3` is a *pre-release*: a Cargo requirement such as
   `winit = "0.30.9"` (and even `winit = "0.31"`) will never select it. Until
   the final release, no stable downstream consumer can get the protocol.

2. **Strongly consider a `0.30.14` backport** of just
   `types/{ext_background_effect,bgr_effects}.rs` + the `state.rs`/`window/*`
   wiring (≈360 lines, see `winit-ext-background-effect.patch` in this repo for
   an exact diff against `0.30.13`).
   - It needs **no dependency changes**: `0.30.13` already requires
     `wayland-protocols = "0.32.8"` with the `staging` feature, and
     `staging/ext-background-effect` has been present since `0.32.8`/`0.32.9`.
   - It is behaviour-preserving for everyone else: `ext-background-effect-v1`
     is preferred, `org_kde_kwin_blur` remains the fallback, X11/Windows/macOS
     untouched.
   - **This single change would light up `window.blur` in Alacritty with zero
     Alacritty code changes**, because Alacritty pins `winit = "0.30.9"`.

3. *(optional)* **Do not silently ignore the `capabilities` event.**
   Both `Dispatch<ExtBackgroundEffectManagerV1>::event` handlers are empty, so
   winit reports blur as accepted even when the compositor answers
   `capabilities` without the `blur` bit (or when the protocol is absent). A
   query (e.g. `Window::blur_supported() -> bool`, or making `set_blur`
   fallible) would let apps like Alacritty log a proper warning instead of
   silently showing transparency only.

4. *(optional)* Also document the new protocol in the `set_blur` platform notes
   for the `0.30.x` docs (`winit/src/window.rs` still says
   *"Wayland: Only works with org_kde_kwin_blur_manager protocol"*).

---

## 2. Report for **alacritty/alacritty**

### Already done (nothing to implement)

The feature is fully plumbed; Alacritty needs **no protocol or blur code**.
Verified in `v0.17.0`:

| file:line | what |
|---|---|
| `alacritty/src/config/window.rs:49` | `window.blur` config field |
| `alacritty/src/display/window.rs:181` | `.with_blur(config.window.blur)` at window creation |
| `alacritty/src/display/window.rs:375` | `set_blur()` → `winit::Window::set_blur` |
| `alacritty/src/window_context.rs:323` | re-applied on live config reload |

`glutin` is **not** a blocker: `glutin` core depends only on
`raw-window-handle` (no winit), and Alacritty does not use `glutin-winit`
(that crate is what pins `winit = "0.30"` / `=0.31.0-beta.3`). Alacritty's
`Cargo.toml` is the only place where the winit version is decided.

### What is still needed

1. **Bump `winit` in `alacritty/Cargo.toml:46`**
   `winit = { version = "0.30.9", ... }` → `0.31`.

2. **Land the 0.31 migration — PR #8750** *"Update to winit-0.31.0-beta.2"*
   (open since 2025-11, ~1.5 k lines across `event.rs`, `display/window.rs`,
   `input/*`, `main.rs`, `window_context.rs`).
   It must be **rebased onto `0.31.0-beta.3`**, because beta.3 is the first
   release that actually contains `ext-background-effect-v1` (beta.2 does not).
   Migration surface per `winit/src/changelog/v0.31.md`:
   - `Window` and `ActiveEventLoop` become traits;
     `ActiveEventLoop::create_window` returns `Box<dyn Window>`
     (`alacritty/src/event.rs:170`, `alacritty/src/display/window.rs:186`)
   - `ApplicationHandler` methods take `&dyn ActiveEventLoop`;
     `user_event` → `user_wake_up`;
     `EventLoopProxy::send_event` → `EventLoopProxy::wake_up`
     (`event.rs`, `window_context.rs`, `ipc.rs`)
   - per-platform `WindowAttributes` structs replace the extension traits
     (`WindowAttributesExtWayland` used in `display/window.rs`)
   - pointer `WindowEvent` overhaul, `DeviceId` → `Option<DeviceId>`,
     surface-size API renames, `Fullscreen::Exclusive(MonitorHandle, VideoMode)`,
     drag-and-drop API rework (hits `input/mod.rs`, `input/keyboard.rs`)

3. **Docs** — nothing in the code, but the documented scope is wrong once the
   bump lands:
   - `extra/man/alacritty.5.scd:139`
     `blur = ... # (works on macOS/KDE Wayland)` → should say macOS, plus any
     Wayland compositor implementing `ext-background-effect-v1`
     (e.g. labwc + labwc-blur) or `org_kde_kwin_blur` (KDE).
   - `CHANGELOG.md` entry under *Added/Changed*.
   - Note for users: blur only shows with `window.opacity < 1`.

### Verification (for whoever reviews the bump)

```sh
WAYLAND_DEBUG=1 alacritty 2>&1 | grep -E "background_effect|set_blur_region"
```

Expected against a compositor serving the protocol:

```
-> wl_registry#2.bind(..., "ext_background_effect_manager_v1", 1, ...)
-> ext_background_effect_manager_v1#N.get_background_effect(new id ext_background_effect_surface_v1#M, wl_surface#S)
-> ext_background_effect_surface_v1#M.set_blur_region(wl_region#R)
```

Stock `alacritty` 0.17.0 prints none of these (it binds only
`org_kde_kwin_blur_manager`, which non-KDE compositors do not advertise).
