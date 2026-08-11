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
  pkgconf \
  fontconfig \
  mpv \
  alsa-lib \
  dbus \
  openssl \
  clang \
  cmake \
  git
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

If you don't have the base development tools:

```bash
sudo pacman -S base-devel
```

## Repository layout

The build expects sibling checkouts under a common parent directory:

```
~/git/
  openmls/             # DWDC fork (v0.8.1 + vault HPKE patches)
  mdk/                 # DWDC fork (upstream + MlsSigner + vault HPKE)
  whitenoise-linux/    # This repo
  keyvault-rs/         # BIP-32 key derivation engine
  wn-kv-test/          # KmLight signer + integration tests
  mdk-e2e-test/        # End-to-end tests
```

Path dependencies in `Cargo.toml` point at `../openmls` and `../mdk`,
so the directory names matter.

## Clone

```bash
cd ~/git
git clone git@github.com:DarkWebDivingClub/openmls.git
git clone git@github.com:DarkWebDivingClub/mdk.git
git clone git@github.com:DarkWebDivingClub/whitenoise-linux.git
```

Add upstream remotes so you can track and sync:

```bash
cd ~/git/openmls && git remote add upstream https://github.com/openmls/openmls
cd ~/git/mdk && git remote add upstream git@github.com:marmot-protocol/mdk.git
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

Check all three repos compile:

```bash
cd ~/git/openmls && cargo check
cd ~/git/mdk && cargo check
cd ~/git/whitenoise-linux && cargo check
```

Run the vault-backed key-package e2e test:

```bash
cd ~/git/mdk-e2e-test && cargo test
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

### Slint UI not found at runtime (debug builds)

Debug builds use Slint's live-reload and look for `.slint` files
relative to the source tree. Run from the repo root:

```bash
cd ~/git/whitenoise-linux && cargo run
```

Release builds embed the UI and work from any directory.
