# alacritty-blur-pkg

Arch Linux packaging for **alacritty 0.17.0** with `ext-background-effect-v1`
blur — the protocol served by
[labwc-blur](https://github.com/grigio/labwc-blur-pkg).

## Why

Alacritty has a `[window] blur = true` option, but the request goes through
**winit**, and winit 0.30 only speaks KDE's legacy `org_kde_kwin_blur_manager`.
labwc-blur serves the standardized `ext-background-effect-v1` instead, so the
request finds no global and is silently dropped: you get the `opacity`, never
the blur.

The support exists upstream on **winit master** (0.31,
`winit-wayland/src/types/{ext_background_effect,bgr_effects}.rs`) — but it is
unreleased, not even in `0.31.0-beta.3` — and alacritty still requires winit
0.30, so simply updating alacritty (0.17.0 is the latest release) cannot help.

This package therefore builds the **unmodified** alacritty 0.17.0 against a
winit 0.30.13 with that support backported, see
`winit-ext-background-effect.patch` (7 files, ~360 lines). The patched winit
prefers `ext-background-effect-v1` and falls back to `org_kde_kwin_blur`, so
KDE/macOS behaviour is unchanged.

## Install

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
```

The blur covers the whole window and is clipped to it by the compositor. With
`opacity = 1.0` there would be nothing to see through.

## Verify

With a labwc session running:

```sh
WAYLAND_DEBUG=1 alacritty 2>&1 | grep -E "background_effect|set_blur_region"
```

Expected on startup:

```
-> wl_registry#2.bind(48, "ext_background_effect_manager_v1", 1, ...)
-> ext_background_effect_manager_v1#16.get_background_effect(new id ext_background_effect_surface_v1#47, wl_surface#41)
-> ext_background_effect_surface_v1#47.set_blur_region(wl_region#48)
```

The stock package prints nothing at all for those greps.

## Contents

| path | what |
|---|---|
| `PKGBUILD` | `alacritty-blur` 0.17.0, stock alacritty built against a patched winit |
| `winit-ext-background-effect.patch` | winit 0.30.13 + `ext-background-effect-v1`, backported from winit master |

## Dropping it

Once winit releases the protocol support and alacritty depends on it,
`sudo pacman -S alacritty` is all that is needed.
