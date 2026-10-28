# alacritty-blur-pkg

[![Build Arch packages](https://github.com/grigio/alacritty-blur-pkg/actions/workflows/build.yml/badge.svg)](https://github.com/grigio/alacritty-blur-pkg/actions/workflows/build.yml)

Arch Linux packaging for **alacritty 0.17.0** with `ext-background-effect-v1`
blur — the standardized protocol served by
[labwc-blur](https://github.com/grigio/labwc-blur-pkg), cosmic-comp, KDE Plasma
6.7+ and other compositors — and with **inertial (kinetic) scrolling**, which
stock alacritty still lacks
([alacritty#4175](https://github.com/alacritty/alacritty/issues/4175)).

## Why

Alacritty has a `[window] blur = true` option, but the request goes through
**winit**, and winit 0.30 only speaks KDE's legacy `org_kde_kwin_blur_manager`.
Every other compositor — labwc-blur, cosmic-comp, anything new — serves the
standardized `ext-background-effect-v1` instead, so the request finds no global
and is silently dropped: you get the `opacity`, never the blur.

The support exists upstream on **winit 0.31 pre-releases**
(`winit-wayland/src/types/{ext_background_effect,bgr_effects}.rs`, first shipped
in `0.31.0-beta.3`, also in `beta.4`) — but there is no final `0.31.0`, cargo
will never select a pre-release for alacritty's `winit = "0.30.9"` requirement,
and the `0.31` migration (PR #8750) has not landed, so simply updating alacritty
(0.17.0 is the latest release) cannot help.

This package therefore builds alacritty 0.17.0 against a winit 0.30.13 with
that support backported, see `winit-ext-background-effect.patch` (7 files,
~360 lines). The patched winit prefers `ext-background-effect-v1` and falls
back to `org_kde_kwin_blur`, so KDE/macOS behaviour is unchanged.

## Compositor support

The package is compositor-agnostic: whatever serves
`ext-background-effect-v1` gets blur, whatever does not stays transparent.

| compositor | works out of the box | notes |
|---|---|---|
| **labwc** + [labwc-blur](https://github.com/grigio/labwc-blur-pkg) | ✅ | what this package was written for |
| **cosmic-comp** (COSMIC ≥ 1.3, tag `epoch-1.3.0`) | ⚠️ after one toggle | needs **Settings → Desktop → Appearance → Frosted glass → Windows**, see below |
| **KDE Plasma ≥ 6.7** | ✅ | served natively |
| **COSMIC < 1.3**, stock labwc/sway/mutter without the protocol | ❌ | transparency only; nothing for the client to bind |

### cosmic-comp: enable "Windows" frosted glass

cosmic-comp advertises `ext_background_effect_manager_v1` with the `blur`
capability to every client, but it only *renders* the effect when the theme
enables it per surface class. In `src/shell/element/window.rs`:

```rust
let frosted = if theme.cosmic().frosted_windows {
    (theme.cosmic().frosted as u8 + 1) as usize
} else {
    0
};
```

and `BlurElement::internal()` returns `Ok(None)` when the strength is `0`. The
default is **off** (`frosted_windows: false` in libcosmic's `Theme::default()`),
which produces exactly the "transparency but no blur" symptom — the client does
everything right and the compositor drops it on the floor.

Turn it on in the GUI (**Settings → Desktop → Appearance → Frosted glass**,
toggle **Windows**; **Frost thickness** sets the strength), or write the keys
directly:

```sh
for t in Dark Light; do
  echo true > ~/.config/cosmic/com.system76.CosmicTheme.$t/v2/frosted_windows
done
```

`cosmic-config` watches those files, so no restart is needed. Note that
`frosted` (the thickness) and `frosted_windows` (the per-surface switch) are
independent: setting the thickness alone still renders nothing.

The same gate is why third-party blur looked broken in
[cosmic-comp#2567](https://github.com/pop-os/cosmic-comp/issues/2567) and
[#2598](https://github.com/pop-os/cosmic-comp/issues/2598) — the latter was
closed as completed with *"It is exposed to clients"*, which is true, but the
theme switch is the part that is easy to miss.

## Why inertial scrolling

Alacritty has no momentum scrolling: a flick across the trackpad stops the
moment the fingers lift, so long output takes swipe after swipe
([alacritty#4175](https://github.com/alacritty/alacritty/issues/4175)).

Upstream is on it — [PR #9008](https://github.com/alacritty/alacritty/pull/9008)
"Add vertical scroll velocity" closes that issue — but it is not merged yet and
is not in 0.17.0. This package carries it as `alacritty-scroll-velocity.patch`
(7 files, +329/−18), a hand-rebased copy of that PR: a new
`[scrolling] velocity` option that keeps the momentum of the last scroll
sequence and applies it frame by frame until new input arrives. The PR's
unrelated format-string cleanups were dropped, its MSRV bump (1.85 → 1.88,
needed for the let-chains it uses) kept, and four small guards added:

- momentum counts as active only from **two samples on**, because with a single
  one (two wheel notches, then stop) nothing would ever clear the flag and the
  window would keep redrawing at the monitor's refresh rate forever;
- `Velocity::direction` is now actually assigned on every non-zero scroll,
  because upstream only set it *inside* the direction-change branch — which
  requires it to already be `Some` — so it stayed `None` forever and a scroll
  in the opposite direction never cancelled the running momentum;
- `Velocity::apply` never ends the momentum on its **first tick** or on a
  **sub-millisecond tick**: the first tick is measured from the last input
  sample (which can be microseconds old when the scroll event itself scheduled
  the redraw), and a tick's delta scales with its interval — so upstream's
  "insignificant delta" check could cancel the momentum before it ever started.
  Whether inertia worked was pure timing luck; with these guards it starts
  reliably and still stops on the first normal frame that moves less than a pixel;
- `Velocity::update` ignores samples that arrive in the **same microsecond**
  (`as_micros()` truncates, so the elapsed time was zero and the division made
  the velocity `inf`) and caps the normalized velocity, while `scroll_terminal`
  rejects non-finite deltas and clamps the line/column count to the grid. Before
  this, an `inf` delta cast to `usize` became `usize::MAX`, the mouse-report and
  alternate-scroll loops ran essentially forever, and the heap grew until
  systemd-oomd killed the whole session (reproduced at ~29 GiB).

## Install

Grab the package from the [latest release](https://github.com/grigio/alacritty-blur-pkg/releases/latest):

```sh
sudo pacman -U alacritty-blur-*.pkg.tar.zst
```

Or build it locally:

```sh
makepkg -sCi
```

It `provides` and `conflicts with` `alacritty`, so pacman swaps it for the
stock package (answer **yes** if it asks). Back to stock:

```sh
sudo pacman -S alacritty
```

## Configure

`~/.config/alacritty/alacritty.toml`:

```toml
[window]
opacity = 0.9   # translucency: what you see through the terminal
blur = true     # ask the compositor to blur behind the window

[scrolling]
velocity = "On" # keep scrolling after a flick (see below)
```

The blur covers the whole window and is clipped to it by the compositor. With
`opacity = 1.0` there would be nothing to see through.

`scrolling.velocity` controls the inertia:

| value | meaning |
|---|---|
| `"Auto"` (default) | momentum for **touch** (touchscreen) only — mice/trackpads usually do their own |
| `"On"` | momentum for every source: flick the trackpad and it keeps scrolling |
| `"Off"` | never — stock alacritty behaviour |

Momentum starts when the fingers leave the surface and stops on the next scroll,
mouse-button or touch event (so `PageDown`, scroll-to-bottom or a click always
win).

## Potential issues

- **Upstream is not final.** [PR #9008](https://github.com/alacritty/alacritty/pull/9008)
  is still open with changes requested (the maintainer wants a different
  friction model), so the feel of the momentum — and possibly the option name —
  may change before this rebases onto whatever lands.
- **The feel is tuned upstream, not here.** The deceleration constant
  (`VELOCITY_DAMPING = 5`, exponential damping) was tuned by the PR author for
  phone and mouse; coasting may start or end sooner/later than Firefox's
  kinetic scrolling on your trackpad. Changing it means rebuilding the package.
- **The patch is not a pure upstream copy.** Besides the rebase, it carries
  four local guards (documented above) that work around real bugs in the PR:
  a permanent redraw loop on two-sample scrolls, direction changes that never
  cancelled momentum (`direction` was dead code), a momentum start that only
  worked if the first redraw happened to land milliseconds after the last input
  sample — likely the "always stops instantly" complaints in the PR discussion —
  and the zero-elapsed/`inf` velocity that could turn a scroll into an unbounded
  loop and leak memory until the session was OOM-killed. CI fails if these guards
  ever disappear from the applied source.
- **`"On"` gives momentum to mouse-wheel notches too**, so a classic wheel
  mouse coasts after a notch. Use the default `"Auto"` (touch only) or `"Off"`
  if that bothers you.
- **Momentum is vertical only** and is killed by any new input — by design,
  but it means a click or keyboard scroll during a coast stops it immediately.
- **Blur is the compositor's call.** The client can only ask: cosmic-comp and
  other compositors may gate the effect behind a per-surface setting
  (`frosted_windows` on COSMIC), and a compositor that implements neither
  `ext-background-effect-v1` nor `org_kde_kwin_blur` leaves the window
  translucent only. See *Compositor support* above.
- **Tested on Wayland/labwc and cosmic-comp** with a laptop trackpad, a
  touchscreen path that only got synthetic-input coverage, and no
  X11/macOS/Windows testing (this package is Arch/Linux only anyway). The
  cosmic-comp test was protocol-level (`capabilities(1)`, `set_blur_region`
  accepted); the visual check was done on labwc.
- The patch bumps the **MSRV to 1.88** (let-chains). No runtime effect; only
  matters if you build with an older Rust.

## Verify

In a session of any compositor serving the protocol (labwc-blur, cosmic-comp,
KDE 6.7+):

```sh
WAYLAND_DEBUG=1 alacritty 2>&1 | grep -E "background_effect|set_blur_region"
```

Expected on startup:

```
-> wl_registry#2.bind(38, "ext_background_effect_manager_v1", 1, ...)
ext_background_effect_manager_v1#16.capabilities(1)
-> ext_background_effect_manager_v1#16.get_background_effect(new id ext_background_effect_surface_v1#46, wl_surface#40)
-> ext_background_effect_surface_v1#46.set_blur_region(wl_region#47)
```

`capabilities(1)` is the `blur` bit: the compositor advertises the feature. It
does so even when a per-surface setting will later drop the effect (cosmic-comp
answers `blur` unconditionally and only honours it when `frosted_windows` is
on), so these lines prove the client side is fine, not that pixels change. If
the `bind` line is missing entirely the compositor does not implement the
protocol, and the stock package prints none of these lines at all.

Beware of clients built against **wayland-protocols 1.45**, which shipped
`EXT_BACKGROUND_EFFECT_MANAGER_V1_CAPABILITY_BLUR` as `0`; such a binary sees no
blur support and silently skips the request. This package is unaffected (it
compiles `wayland-protocols` 0.32.9 from the lock file), but a distro package
built in that window may not be.

## Contents

| path | what |
|---|---|
| `PKGBUILD` | `alacritty-blur` 0.17.0, alacritty + both patches, built against a patched winit |
| `winit-ext-background-effect.patch` | winit 0.30.13 + `ext-background-effect-v1`, backported from winit master |
| `alacritty-scroll-velocity.patch` | alacritty + `scrolling.velocity` inertial scrolling (PR #9008, closes #4175) |
| `.github/workflows/build.yml` | CI: builds in an `archlinux:base-devel` container, checks the protocol markers, the inertial-scrolling markers and the local guards, uploads the artifact and publishes/refreshes a `v<pkgver>-<pkgrel>` release |

## CI

The workflow runs on pushes that touch `PKGBUILD`, either patch or the
workflow itself (and on manual dispatch), inside an `archlinux:base-devel`
container. It runs `makepkg -sCi` as an unprivileged `builder`, then fails the
job if the packaged binary lacks `ext_background_effect_manager_v1`,
`get_background_effect` or `set_blur_region`, if the KDE
`org_kde_kwin_blur` fallback is gone, if the man page does not document
`scrolling.velocity`, if the packaged binary has no `Config error: velocity`
config field (i.e. the scroll-velocity patch was not compiled in), if the
applied `event.rs` no longer carries the five local-guard comments on top of
PR #9008 (redraw loop, direction tracking, racy first tick, zero-elapsed /
inf velocity), or if the package
does not `provides`/`conflicts with` `alacritty`. The packages land both as an
artifact and as the assets of a GitHub release tagged `v<pkgver>-<pkgrel>` (a
rebuild of the same version refreshes them).

## Dropping it

Once winit releases the protocol support and alacritty depends on it — and
once PR #9008 lands in a release — `sudo pacman -S alacritty` is all that is
needed.
