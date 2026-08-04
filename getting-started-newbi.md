# Getting Started with White Noise

White Noise is an end-to-end encrypted group messenger built on Nostr
and the MLS protocol (RFC 9420). All messages are end-to-end
encrypted — the relay sees only opaque ciphertext.

White Noise uses KeyMaster Avatar for key management. Your private
keys live on your phone (or in a local KeyMaster daemon) and are
never stored on the desktop. The app talks to the signing agent over
a local Unix socket.

## 1. Set up KeyMaster Avatar

Follow the KeyMaster Avatar getting-started guide to install and
configure the signing agent:

[Getting Started with KeyMaster Avatar](https://github.com/DarkWebDivingClub/club.dwdc.keymaster.avatar/blob/master/getting-started-newbi.md)

The guide covers two options:

- **Phone-based** (recommended) — keys stay on your Android phone,
  the desktop signs remotely via a Nostr relay
- **Host-based** — keys are derived from a BIP-39 seed phrase stored
  on the desktop using `keymaster-desktop` and `keyvault-cli`

Complete through at least **step 10** (environment variables) so that
`NOSTR_SA_SOCK` points to the signing-agent socket.

## 2. Install White Noise

If you have not yet added the DWDC APT repository (the avatar guide
does this in step 2), add it now:

```bash
curl -fsSL https://apt.dwdc.club/dwdc-apt-repo.gpg \
  | sudo tee /usr/share/keyrings/dwdc-apt.gpg > /dev/null

DISTRO=$(lsb_release -cs)
echo "deb [signed-by=/usr/share/keyrings/dwdc-apt.gpg] https://apt.dwdc.club $DISTRO alfa" \
  | sudo tee /etc/apt/sources.list.d/dwdc.list

sudo apt update
```

Install the package:

```bash
sudo apt install whitenoise-linux
```

## 3. Launch

```bash
whitenoise-linux
```

If `NOSTR_SA_SOCK` is set (from the avatar guide step 10), the app
connects to the signing agent automatically. You should see your
KeyMaster identity appear — no password prompt, no key generation.

If `NOSTR_SA_SOCK` is not set, the app falls back to standalone
mode with a local password-encrypted key vault.

## 4. Connect to the relay

1. Open **Settings > Relays**
2. Add `wss://relay.dwdc.club`
3. The app publishes your MLS key package to the relay automatically

Your key package contains only public keys. Other users on the same
relay can now invite you to encrypted groups.

## 5. Join or create a group

- **Create a group**: Click "+", name the group, and add members by
  their Nostr npub
- **Accept an invite**: Incoming MLS Welcome messages are processed
  automatically — the group appears in your chat list

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No identity appears on launch | Verify `echo $NOSTR_SA_SOCK` prints a path and `ls $NOSTR_SA_SOCK` shows a socket file |
| "sa-daemon connect: Connection refused" | Restart the signing agent: `systemctl --user restart km-nostr-sa` |
| Can't reach relay | Check that `wss://relay.dwdc.club` is not blocked by firewall |
| App won't start | Ensure libmpv, libfontconfig, and libasound2 are installed (`apt install` pulls them automatically) |
| "Failed to open connection to X server" | White Noise requires a graphical desktop (X11 or Wayland) |

## System requirements

- Debian 13 (Trixie) or Ubuntu 26.04 (Resolute), amd64
- ~120 MB disk space
- Graphical desktop (X11 or Wayland)
- KeyMaster Avatar set up and running (see step 1)
- Network access to `wss://relay.dwdc.club`
