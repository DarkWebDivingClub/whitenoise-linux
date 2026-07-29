# Getting Started — Developer

This guide covers setting up the DWDC fork of whitenoise-linux for
local development, including the keyvault-backed identity path.

## Prerequisites

- Rust 1.85+ (edition 2024)
- System libraries:

```bash
# Debian / Ubuntu
sudo apt install pkg-config libmpv-dev libfontconfig-dev \
  libasound2-dev libdbus-1-dev
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

## Background: How keyvault key derivation works

All cryptographic identity in White Noise is derived from a single
BIP-39 mnemonic (24 words = 256 bits of entropy). The `keyvault-rs`
crate implements a BIP-32 hierarchical deterministic derivation engine
that produces protocol-specific keys at well-defined paths. No private
key is ever stored — keys are re-derived from the mnemonic on every
launch.

### The derivation tree

```
mnemonic (24 words)
  │
  └─ BIP-39 seed (512 bits via PBKDF2)
       │
       └─ BIP-32 master key + chain code (HMAC-SHA512, key="Bitcoin seed")
            │
            ├─ m/44'/1237'/<id>'/<alg>'/<cfg>'     Nostr (secp256k1 Schnorr)
            ├─ m/44'/1242'/<id>'/<alg>'/<cfg>'     MLS (Ed25519, X25519)
            ├─ m/44'/1238'/...                     SSH
            ├─ m/44'/1239'/...                     OpenPGP
            └─ ...                                 (other protocols)
```

All derivation is **hardened-only** (every index has bit 31 set). The
coin-type constants follow a protocol registry:

| Protocol | Coin type |
|----------|-----------|
| Nostr    | 1237      |
| SSH      | 1238      |
| OpenPGP  | 1239      |
| X.509    | 1240      |
| WireGuard| 1241      |
| MLS      | 1242      |

### Path structure (5 levels)

```
m / purpose' / coin_type' / identity' / algorithm' / config'
    44          1237|1242    mangle(email)  see below   see below
```

**Level 3 — Identity**: `mangle(email)` produces a deterministic
31-bit index from the identity string (first 4 bytes of SHA-256,
masked to 31 bits). Two users with different emails get different
subtrees from the same seed.

**Level 4 — Algorithm field** (31 bits, packed):

```
bits 30-16: algorithm (15 bits)    0=Schnorr, 1=Ed25519
bits 15-8:  variant   (8 bits)     usually 0
bits  7-0:  role      (8 bits)     0=sign, 1=HPKE
```

**Level 5 — Config field** (31 bits, packed):

```
bits 30-24: csprng mode (7 bits)   0=none (deterministic)
bits 23-0:  index       (24 bits)  key counter
```

### Keys derived for White Noise

Given a mnemonic and `DM_EMAIL="alice@example.com"`:

```
mangled = mangle("alice@example.com") = 0x748b0a69

Nostr identity (secp256k1 Schnorr):
  m/44'/1237'/0x748b0a69'/0x00000000'/0x00000000'
  → 32-byte seed → BIP-340 x-only pubkey

MLS signing (Ed25519):
  m/44'/1242'/0x748b0a69'/0x00010000'/0x00000000'
  → 32-byte seed → Ed25519 keypair

HPKE init key #0 (X25519):
  m/44'/1242'/0x748b0a69'/0x00010001'/0x00000000'
  → 32-byte seed → X25519 keypair

HPKE init key #1 (X25519):
  m/44'/1242'/0x748b0a69'/0x00010001'/0x00000001'
  → next key package uses the next index
```

The algorithm field `0x00010001` means `alg=Ed25519(1), variant=0,
role=HPKE(1)`. Even though the role says "HPKE", the underlying
derivation produces a 32-byte seed that is used as an X25519 private
key (Curve25519 and Ed25519 share the same scalar field).

### HPKE key rotation

Each MLS key package publishes a fresh X25519 init public key. The
config field's 24-bit index increments for each new key package. When
recovering on a new device, the signer scans indices 0..N until it
finds the public key matching the incoming Welcome's KEM output, then
performs `X25519(private, kem_output)` to decapsulate.

### The vault interface

`keyvault-rs` exposes a single entry point:

```rust
trait KeyVault {
    fn execute(
        &self,
        function: u32,    // 0=export_seed, 1=get_pubkey, 2=sign, 3=key_agreement
        payload: &[u8],   // data to sign, or peer pubkey for DH
        path: &[u32],    // 5-level BIP-32 path (all hardened)
    ) -> Result<Vec<u8>, VaultError>;
}
```

Functions:

| ID | Name | Returns |
|----|------|---------|
| 0 | `FN_EXPORT_SEED` | Raw 32-byte leaf key |
| 1 | `FN_GET_PUBLIC_KEY` | Public key (algorithm-aware) |
| 2 | `FN_SIGN` | Signature over payload |
| 3 | `FN_KEY_AGREEMENT` | Shared secret (X25519 DH or secp256k1 ECDH) |

The vault never exposes private keys to callers outside the crate.
`FN_EXPORT_SEED` exists for the Nostr bootstrap path (which needs
the raw scalar to construct a `nostr::Keys`), but MLS operations use
only `FN_SIGN` and `FN_KEY_AGREEMENT`.

### KmLight: the in-process key manager

`wn-kv-test::km_light::KmLight` wraps the vault and implements the
`MlsSigner` trait that mdk's cgka-engine calls:

```
KmLight::new(mnemonic, vec!["alice@example.com"])
  → derives Nostr pubkey at m/44'/1237'/<mangle>'/.../
  → caches account list

MlsSigner::get_mls_pubkey(nostr_pk, device_id, ciphersuite)
  → derives Ed25519 pubkey at m/44'/1242'/<mangle>'/.../

MlsSigner::mls_sign(mls_pk, data)
  → signs with Ed25519 at the cached path

MlsSigner::mls_next_hpke_init_key(mls_pk, current_pk)
  → scans indices until it finds current, returns index+1

MlsSigner::mls_hpke_decap(hpke_pk, kem_output)
  → scans indices to match pubkey, performs X25519 DH
```

This is the same interface a future hardware-backed signer or
NIP-46 remote signer would implement.

## Run with keyvault identity

The app derives all keys from a BIP-39 mnemonic when two environment
variables are set:

```bash
export DM_MNEMONIC="abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about"
export DM_EMAIL="alice@example.com"
cargo run
```

This bypasses the password/vault login screen entirely. The mnemonic
deterministically produces:

- A Nostr secp256k1 identity (signs events, identifies you on relays)
- An Ed25519 MLS signing key (signs MLS handshake messages)
- X25519 HPKE init keys (decrypt MLS Welcome messages)

The email is an identity label used in the BIP-32 derivation path
(mangled to a 32-bit index). It never leaves the machine.

For a production-like 24-word seed, generate one with any BIP-39 tool
and use it in place of the test mnemonic above.

## Run without keyvault (legacy mode)

Without `DM_MNEMONIC`, the app falls back to the password-encrypted
vault flow: generate or import an nsec, set a local password. This is
the upstream-stock behaviour.

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

## Building a .deb

The `packaging/ubuntu/resolute` branch carries Debian packaging on
top of master:

```bash
git checkout packaging/ubuntu/resolute
dpkg-buildpackage -b -us -uc
```

The `.deb` lands in the parent directory.

## Fork structure

Each DWDC fork carries patches on top of an upstream release:

| Repo | Base | Patches |
|------|------|---------|
| openmls | `openmls-v0.8.1` tag | KeyPackageBuilder::init_keypair, HpkeKeyPurpose |
| mdk | `upstream/master` | MlsSigner trait, vault HPKE provider |
| whitenoise-linux | `upstream/master` | Path deps, vault HPKE wiring, icons |

Each has a `FORK.md` at root. The remote pushing to GitHub is named
`dwdc`.

## Syncing with upstream

1. `git fetch upstream`
2. `git reset --hard upstream/master` (or the compatible tag)
3. Cherry-pick our patches
4. Re-add `FORK.md`
5. `git push --force-with-lease dwdc <branch>`
6. Verify: `cargo check` on all three, `cargo test` in mdk-e2e-test

## Key source files

| Path | Role |
|------|------|
| `src/backend.rs` | `keyvault_identity()` — mnemonic-to-signer boot path |
| `src/vault.rs` | Password-encrypted secret storage (legacy path) |
| `src/wiring/panes.rs` | Login/unlock UI callbacks |
| `Cargo.toml` | Path deps to `../openmls` and `../mdk` |

## Environment variables

| Variable | Effect |
|----------|--------|
| `DM_MNEMONIC` | BIP-39 mnemonic — enables keyvault identity derivation |
| `DM_EMAIL` | Identity label for key derivation (never sent anywhere) |
| `DM_HOME` | Data directory override (vault, media cache, MLS state) |
| `RUST_LOG` | tracing filter (default: `info`) |
