# Maintainer: local build of alacritty with ext-background-effect-v1 blur
#
# Upstream alacritty 0.17.0 has a [window] blur option, but the request goes
# through winit, and winit 0.30 only implements KDE's org_kde_kwin_blur_manager.
# labwc-blur serves the standardized ext-background-effect-v1 protocol instead,
# so the request found no global and was silently dropped (transparency only).
#
# This package builds the unmodified alacritty 0.17.0 against a winit 0.30.13
# with the ext-background-effect-v1 support backported from winit master
# (winit-wayland/src/types/{ext_background_effect,bgr_effects}.rs), preferring
# ext-background-effect-v1 and falling back to org_kde_kwin_blur.
#
# Drop this package with `pacman -S alacritty` to return to the stock build
# once winit releases the support (winit >= 0.31) and alacritty depends on it.

pkgname=alacritty-blur
pkgver=0.17.0
pkgrel=1
pkgdesc='A cross-platform, GPU-accelerated terminal emulator (blur via ext-background-effect-v1)'
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
)
# The .crate is extracted by prepare() so the patch can be applied to it.
noextract=('winit-0.30.13.crate')
sha256sums=(
  '38d6527d346cda5c6049332a1f3338a89ea66cd7981b54d4c3ce801b392496f8'
  'a6755fa58a9f8350bd1e472d4c3fcc25f824ec358933bba33306d0b63df5978d'
  '173b96ab33e48944ef1b420d31d1b6a7c9d897f6d5105be19088b3605ac8cfed'
)

prepare() {
  # Build the patched winit the alacritty workspace will resolve to.
  mkdir -p winit
  tar -xf winit-0.30.13.crate -C winit --strip-components=1
  patch -d winit -p1 < winit-ext-background-effect.patch

  cd alacritty-${pkgver}
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
