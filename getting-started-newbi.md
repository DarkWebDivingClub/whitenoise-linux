# Getting Started with White Noise

White Noise is an end-to-end encrypted group messenger built on Nostr
and the MLS protocol. All your keys — Nostr identity, MLS signing,
and HPKE encryption — are derived deterministically from a single
24-word seed phrase. No private key material is ever stored on disk
or sent over the network.

## 1. Add the APT repository

```bash
curl -fsSL https://apt.dwdc.club/dwdc-apt-repo.gpg \
  | sudo tee /usr/share/keyrings/dwdc-apt.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/dwdc-apt.gpg] https://apt.dwdc.club alfa main" \
  | sudo tee /etc/apt/sources.list.d/dwdc.list

sudo apt update
```

## 2. Install White Noise

```bash
sudo apt install whitenoise-linux
```

## 3. Generate your seed phrase

Your 24-word seed is the root of your entire identity. Generate it
**offline** using a trusted BIP-39 tool, then store it on paper in a
safe place.

If you have a hardware wallet or an existing BIP-39 seed you want to
reuse, you can use those same 24 words.

To generate a new seed from the command line:

```bash
# Option A: use an existing tool (e.g. from your password manager or
# hardware wallet setup wizard)

# Option B: generate with openssl + a BIP-39 wordlist
#   (256 bits of entropy = 24 words)
python3 -c "
from secrets import token_bytes
from hashlib import sha256

wordlist = open('/usr/share/dict/bip39-english.txt').read().split()
entropy = token_bytes(32)
h = sha256(entropy).digest()
bits = bin(int.from_bytes(entropy, 'big'))[2:].zfill(256)
bits += bin(h[0])[2:].zfill(8)  # checksum
words = [wordlist[int(bits[i:i+11], 2)] for i in range(0, 264, 11)]
print(' '.join(words))
"
```

Or simply use any BIP-39 compatible wallet (Sparrow, Electrum,
Trezor Suite, Coldcard) to generate 24 words.

**Important:**
- Write the 24 words on paper. Do not store them digitally.
- The seed can recover your identity on any device, forever.
- Anyone with your seed controls your identity. Guard it accordingly.

## 4. Launch White Noise with your seed

Set your seed and identity label, then start the app:

```bash
export DM_MNEMONIC="your twenty four words go here in order separated by spaces"
export DM_EMAIL="you@example.com"
whitenoise-linux
```

The email is used as your identity label for key derivation — it
determines which Nostr keypair and MLS keys are derived from the
seed. It is never sent to any server.

The app derives all keys on startup and bypasses the password screen.
Nothing is written to disk unencrypted.

### Making it permanent

Add to `~/.config/whitenoise/env` (create if needed):

```bash
DM_MNEMONIC="your twenty four words go here in order separated by spaces"
DM_EMAIL="you@example.com"
```

Then create a wrapper script or systemd drop-in that sources this file
before launching.

## 5. Connect to the relay

1. Open **Settings > Relays**
2. Add `wss://relay.dwdc.club`
3. The app publishes your MLS key package to the relay automatically

Your key package contains only public keys — the seed never leaves
your machine. Other users on the relay can now invite you to groups.

## 6. Join or create a group

- **Create a group**: Click "+", name the group, and add members by
  their Nostr npub
- **Accept an invite**: Incoming MLS Welcome messages are processed
  automatically — the group appears in your chat list

All messages are end-to-end encrypted with MLS (RFC 9420). The relay
sees only opaque ciphertext.

## How key derivation works

From your 24-word seed, the app deterministically derives:

| Key | Derivation path | Purpose |
|-----|-----------------|---------|
| Nostr identity (secp256k1) | `m/44'/1237'/<id>'/...` | Signs Nostr events, identifies you on relays |
| MLS signing (Ed25519) | `m/44'/1242'/<id>'/...` | Signs MLS handshake messages |
| HPKE init (X25519) | `m/44'/1242'/<id>'/...` | Decrypts MLS Welcome messages |

Because all keys are derived from the seed, you can restore your full
identity on a new device with just the 24 words and your email label.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "KmLight::new failed" | Check that `DM_MNEMONIC` contains exactly 24 valid BIP-39 English words |
| "no accounts" | Verify `DM_EMAIL` is set |
| Can't reach relay | Check that `wss://relay.dwdc.club` is not blocked by firewall |
| App won't start | Ensure libmpv, libfontconfig, and libasound2 are installed |

## System requirements

- Debian 12+ / Ubuntu 24.04+ (amd64)
- ~60 MB disk space
- Network access to `wss://relay.dwdc.club`
