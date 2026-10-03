# Maintainer: local build of alacritty with ext-background-effect-v1 blur and
# inertial scrolling
#
# Upstream alacritty 0.17.0 has a [window] blur option, but the request goes
# through winit, and winit 0.30 only implements KDE's org_kde_kwin_blur_manager.
# labwc-blur serves the standardized ext-background-effect-v1 protocol instead,
# so the request found no global and was silently dropped (transparency only).
#
# This package builds alacritty 0.17.0 with two local patches:
#
#  1. winit-ext-background-effect.patch — backports the ext-background-effect-v1
#     support of winit master (winit-wayland/src/types/{ext_background_effect,
#     bgr_effects}.rs) into the winit 0.30.13 this workspace builds against,
#     preferring ext-background-effect-v1 and falling back to org_kde_kwin_blur.
#  2. alacritty-scroll-velocity.patch — alacritty PR #9008 "Add vertical scroll
#     velocity" (closes #4175): the `scrolling.velocity` option that gives
#     inertial/kinetic scrolling, so a trackpad flick or touchscreen swipe keeps
#     scrolling. Hand-rebased onto 0.17.0 (drops the PR's unrelated format-string
#     cleanups); bumps the MSRV to 1.88 because of the let-chains it uses, and
#     adds three guards:
#       - `Velocity::is_active` requires two samples, since a single one (two wheel
#         notches, then stop) would keep requesting redraws at the monitor refresh
#         rate forever.
#       - `Velocity::direction` is assigned on every non-zero scroll (upstream only
#         assigned it inside the direction-change branch, which requires it to
#         already be `Some`, so it was never set and direction changes never
#         cancelled the momentum).
#       - `Velocity::apply` never ends the momentum on its first tick or on a
#         sub-millisecond tick: the first tick is measured from the last input
#         sample (microseconds old when the scroll event scheduled the redraw) and
#         a tick's delta scales with its interval, so upstream's "insignificant
#         delta" check could cancel the momentum before it ever started — momentum
#         start was pure timing luck.
#
# Drop this package with `pacman -S alacritty` to return to the stock build,
# once winit releases the blur support (winit >= 0.31) and alacritty ships the
# scroll velocity patch itself.

pkgname=alacritty-blur
pkgver=0.17.0
pkgrel=2
pkgdesc='A cross-platform, GPU-accelerated terminal emulator (blur via ext-background-effect-v1, inertial scrolling)'
url="https://github.com/alacritty/alacritty"
arch=('x86_64')
license=('Apache-2.0' 'MIT')
provides=("alacritty=${pkgver}")
conflicts=('alacritty')
depends=(
  freetype2
  fontconfig
  libxi
  libxcursor
  libxkbcommon
  libxkbcommon-x11
  libxrandr
)
makedepends=(
  cargo
  cmake
  desktop-file-utils
  git
  libxcb
  ncurses
  rust
  scdoc
)
optdepends=('ncurses: for alacritty terminfo database')
source=(
  "https://github.com/alacritty/alacritty/archive/refs/tags/v${pkgver}.tar.gz"
  "https://static.crates.io/crates/winit/winit-0.30.13.crate"
  "winit-ext-background-effect.patch"
  "alacritty-scroll-velocity.patch"
)
# The .crate is extracted by prepare() so the patch can be applied to it.
noextract=('winit-0.30.13.crate')
sha256sums=(
  '38d6527d346cda5c6049332a1f3338a89ea66cd7981b54d4c3ce801b392496f8'
  'a6755fa58a9f8350bd1e472d4c3fcc25f824ec358933bba33306d0b63df5978d'
  '173b96ab33e48944ef1b420d31d1b6a7c9d897f6d5105be19088b3605ac8cfed'
  '1bf47bc7d0a4ecf7a80055ca312aade81a9f085bf526ee6dcf930aa1314bcbf1'
)

prepare() {
  # Build the patched winit the alacritty workspace will resolve to.
  # Rebuilt from scratch: the ext-background-effect patch creates files, which
  # would make a second run of prepare() fail on the leftovers of the first.
  rm -rf winit
  mkdir -p winit
  tar -xf winit-0.30.13.crate -C winit --strip-components=1
  patch -d winit -p1 < winit-ext-background-effect.patch

  cd alacritty-${pkgver}
  # Inertial scrolling: alacritty PR #9008 (closes #4175), `scrolling.velocity`.
  patch -p1 < ../alacritty-scroll-velocity.patch

  printf '\n# Backport of ext-background-effect-v1 blur support into winit 0.30\n[patch.crates-io]\nwinit = { path = "%s" }\n' \
    "$srcdir/winit" >> Cargo.toml
  # Resolves the path patch into Cargo.lock (wayland-protocols 0.32.9, which
  # the lock already pins, ships the ext-background-effect-v1 bindings) and
  # downloads the crates. Not --locked: the lock file has to change for the
  # [patch.crates-io] entry.
  cargo fetch --target "$(rustc --print host-tuple)"
}

build() {
  cd alacritty-${pkgver}
  CARGO_INCREMENTAL=0 cargo build --release --offline
}

package() {
  cd alacritty-${pkgver}
  desktop-file-install -m 644 --dir "$pkgdir/usr/share/applications/" "extra/linux/Alacritty.desktop"
  install -D -m755 "target/release/alacritty" "$pkgdir/usr/bin/alacritty"
  scdoc < "extra/man/alacritty.1.scd" | install -D -m644 /dev/stdin \
          "$pkgdir/usr/share/man/man1/alacritty.1"
  scdoc < "extra/man/alacritty.5.scd" | install -D -m644 /dev/stdin \
          "$pkgdir/usr/share/man/man5/alacritty.5"
  scdoc < "extra/man/alacritty-msg.1.scd" | install -D -m644 /dev/stdin \
          "$pkgdir/usr/share/man/man1/alacritty-msg.1"
  scdoc < "extra/man/alacritty-bindings.5.scd" | install -D -m644 /dev/stdin \
          "$pkgdir/usr/share/man/man5/alacritty-bindings.5"
  scdoc < "extra/man/alacritty-escapes.7.scd" | install -D -m644 /dev/stdin \
          "$pkgdir/usr/share/man/man7/alacritty-escapes.7"
  install -D -m644 "extra/linux/org.alacritty.Alacritty.appdata.xml" "$pkgdir/usr/share/metainfo/org.alacritty.Alacritty.appdata.xml"
  install -D -m644 "extra/completions/alacritty.bash" "$pkgdir/usr/share/bash-completion/completions/alacritty"
  install -D -m644 "extra/completions/_alacritty" "$pkgdir/usr/share/zsh/site-functions/_alacritty"
  install -D -m644 "extra/completions/alacritty.fish" "$pkgdir/usr/share/fish/vendor_completions.d/alacritty.fish"
  install -D -m644 "extra/logo/compat/alacritty-term.svg" "$pkgdir/usr/share/pixmaps/Alacritty.svg"
}
