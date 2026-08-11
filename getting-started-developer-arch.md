# Getting Started — Developer (Arch Linux)

This guide covers setting up the DWDC fork of whitenoise-linux for
local development on Arch Linux, including the keyvault-backed
identity path.

For background on key derivation, fork structure, and environment
variables see [getting-started-developer.md](getting-started-developer.md).

## Prerequisites

### Rust toolchain

Install via `rustup` (recommended) or the Arch package:

```bash
# Option A: rustup (recommended — matches the edition-2024 requirement)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup default stable    # 1.85+

# Option B: Arch package (check version is >= 1.85)
sudo pacman -S rust
```

### System libraries

```bash
sudo pacman -S \
  base-devel \
  pkgconf \
  fontconfig \
  mpv \
  alsa-lib \
  dbus \
  openssl \
  clang \
  cmake \
  git \
  librsvg
```

Arch package names differ from Debian — here is the mapping:

| Debian / Ubuntu | Arch | Purpose |
|-----------------|------|---------|
| `pkg-config` | `pkgconf` | Build-time pkg-config |
| `libfontconfig-dev` | `fontconfig` | Font configuration |
| `libmpv-dev` | `mpv` | Media playback |
| `libasound2-dev` | `alsa-lib` | ALSA audio |
| `libdbus-1-dev` | `dbus` | D-Bus IPC |
| `libssl-dev` | `openssl` | TLS (needed by some crates) |
| `build-essential` | `base-devel` | Compiler toolchain |
| `librsvg2-bin` | `librsvg` | SVG-to-PNG icon rendering |

## Repository layout

The build expects five sibling checkouts under a common parent
directory. The directory names must match because `Cargo.toml` uses
relative path dependencies (`../openmls`, `../mdk`, etc.):

```
~/git/
  openmls/             # DWDC fork (v0.8.1 + vault HPKE patches)
  mdk/                 # DWDC fork (upstream + MlsSigner + vault HPKE)
  whitenoise-linux/    # This repo
  keyvault-rs/         # BIP-32 key derivation engine
  wn-kv-test/          # sa-client, sa-daemon, KmLight
```

## Clone

Use HTTPS URLs (not SSH) unless you have SSH keys for GitHub:

```bash
cd ~/git
git clone https://github.com/DarkWebDivingClub/openmls.git
git clone https://github.com/DarkWebDivingClub/mdk.git
git clone https://github.com/DarkWebDivingClub/whitenoise-linux.git
git clone https://github.com/DarkWebDivingClub/keyvault-rs.git
git clone https://github.com/DarkWebDivingClub/wn-kv-test.git
```

Add upstream remotes so you can track and sync:

```bash
cd ~/git/openmls && git remote add upstream https://github.com/openmls/openmls
cd ~/git/mdk && git remote add upstream https://github.com/marmot-protocol/mdk.git
cd ~/git/whitenoise-linux && git remote add upstream https://github.com/marmot-protocol/whitenoise-linux.git
```

## Build

```bash
cd ~/git/whitenoise-linux
cargo build
```

First build fetches crates and compiles the full tree including the
generated Slint UI module (~3 min). After that, incremental Rust edits
rebuild in seconds.

Release build:

```bash
cargo build --release
```

## Building with makepkg (PKGBUILD)

Standard `makepkg` expects a single source tree. WhiteNoise needs
five sibling repos due to Cargo path dependencies. The PKGBUILD
below handles this by cloning all repos into `$srcdir` side by side
during `prepare()`.

Save this as `PKGBUILD` in an empty directory:

```bash
# Maintainer: Your Name <you@example.com>
pkgname=whitenoise-dwdc-linux-git
pkgver=0.1.0
pkgrel=1
pkgdesc="End-to-end encrypted group chat over Nostr (DWDC fork)"
arch=('x86_64')
url="https://github.com/DarkWebDivingClub/whitenoise-linux"
license=('AGPL-3.0-or-later')
depends=('fontconfig' 'mpv' 'alsa-lib' 'dbus')
makedepends=('rust' 'cargo' 'pkgconf' 'clang' 'cmake' 'git' 'librsvg')
provides=('whitenoise-linux')
conflicts=('whitenoise-linux')

# All five repos are listed as sources so makepkg tracks them,
# but we clone manually in prepare() to control directory names.
source=(
  'whitenoise-linux::git+https://github.com/DarkWebDivingClub/whitenoise-linux.git'
  'openmls::git+https://github.com/DarkWebDivingClub/openmls.git'
  'mdk::git+https://github.com/DarkWebDivingClub/mdk.git'
  'keyvault-rs::git+https://github.com/DarkWebDivingClub/keyvault-rs.git'
  'wn-kv-test::git+https://github.com/DarkWebDivingClub/wn-kv-test.git'
)
sha256sums=('SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP')

pkgver() {
  cd "$srcdir/whitenoise-linux"
  git describe --tags --long 2>/dev/null | sed 's/^v//;s/-/.r/;s/-/./g' \
    || echo "$pkgver"
}

build() {
  cd "$srcdir/whitenoise-linux"
  export CARGO_TARGET_DIR="$srcdir/target"
  cargo build --release --no-default-features
}

package() {
  install -Dm755 "$srcdir/target/release/whitenoise-linux" \
    "$pkgdir/usr/bin/whitenoise-linux"

  cd "$srcdir/whitenoise-linux"

  # Desktop file
  install -Dm644 assets/whitenoise-linux.desktop \
    "$pkgdir/usr/share/applications/whitenoise-linux.desktop"

  # Icons (render PNGs from SVG)
  for size in 48 128 256; do
    rsvg-convert -w $size -h $size assets/svg/logo.svg \
      -o "$srcdir/whitenoise-linux-${size}.png"
    install -Dm644 "$srcdir/whitenoise-linux-${size}.png" \
      "$pkgdir/usr/share/icons/hicolor/${size}x${size}/apps/whitenoise-linux.png"
  done
  install -Dm644 assets/svg/logo.svg \
    "$pkgdir/usr/share/icons/hicolor/scalable/apps/whitenoise-linux.svg"

  # License
  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
```

Build with:

```bash
makepkg -si
```

The key points:

- **HTTPS URLs** — avoids SSH key issues with GitHub.
- **All five repos as git sources** — makepkg clones them as siblings
  under `$srcdir/`, which is exactly the layout Cargo's path
  dependencies expect.
- **`--no-default-features`** — disables Slint live-reload so the
  UI is compiled into the binary (self-contained, no runtime file
  lookups).
- **`CARGO_TARGET_DIR`** — keeps build artifacts outside the source
  trees to avoid confusing makepkg.

## Run with keyvault identity

The app derives all keys from a BIP-39 mnemonic when two environment
variables are set:

```bash
export DM_MNEMONIC="abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about"
export DM_EMAIL="alice@example.com"
cargo run
```

This bypasses the password/vault login screen entirely. See
[getting-started-developer.md](getting-started-developer.md) for
details on the derivation tree and what these keys produce.

## Run with signing-agent daemon

For daemon-backed signing (no private key in the WhiteNoise process):

```bash
# Build the daemon
cd ~/git/wn-kv-test && cargo build -p sa-daemon

# Start in one terminal
DM_MNEMONIC="abandon ..." DM_EMAILS="alice@atlanta.com" \
  ~/git/wn-kv-test/target/debug/sa-daemon

# In another terminal, export the socket path and run
export NOSTR_SA_SOCK=<path printed by sa-daemon>
cd ~/git/whitenoise-linux && cargo run
```

## Run without keyvault (legacy mode)

Without `DM_MNEMONIC`, the app falls back to the password-encrypted
vault flow: generate or import an nsec, set a local password.

```bash
cargo run
```

## Tests

Check all repos compile:

```bash
cd ~/git/openmls && cargo check
cd ~/git/mdk && cargo check
cd ~/git/whitenoise-linux && cargo check
```

Run the signing-agent integration test:

```bash
cd ~/git/wn-kv-test && cargo build -p sa-daemon && cargo test -p sa-client
```

## Git hooks

Install the project hooks before your first commit:

```bash
cd ~/git/whitenoise-linux
scripts/install-hooks.sh
```

The pre-commit hook enforces `cargo fmt` and `cargo clippy` gates.

## Troubleshooting

### `could not find system library 'fontconfig'`

Install the library and ensure `pkg-config` can find it:

```bash
sudo pacman -S fontconfig pkgconf
```

### `failed to run custom build command for 'mpv-sys'`

Install mpv development files:

```bash
sudo pacman -S mpv
```

Arch ships headers with the main package (no separate `-dev`).

### `openssl-sys: Could not find directory of OpenSSL installation`

```bash
sudo pacman -S openssl
export OPENSSL_DIR=/usr
```

### `failed to get manifest for dependency` / path dep not found

All five repos must be cloned as siblings with the exact directory
names listed in the Repository Layout section. If any repo is missing
or misnamed, Cargo will fail to resolve path dependencies.

### Slint UI not found at runtime (debug builds)

Debug builds use Slint's live-reload and look for `.slint` files
relative to the source tree. Run from the repo root:

```bash
cd ~/git/whitenoise-linux && cargo run
```

Release builds (and `makepkg` builds with `--no-default-features`)
embed the UI and work from any directory.
