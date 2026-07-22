# apt — Wondersmith APT package repository & control plane

A GPG-signed [Debian APT repository](https://the-wondersmith.github.io/apt) served via GitHub Pages, plus the **control plane** that
onboards and feeds package repos into it.

|                  |                                         |
|:-----------------|:---------------------------------------:|
| **Distribution** |          `trixie` (Debian 13)           |
| **Component**    |                 `main`                  |
| **Arch**         |                  `all`                  |
| **Install**      | see [Installation](#installation) below |

This repository does two jobs:

1. **APT repository host.** A stateless, `apt-ftparchive`-generated index over a durable package **pool** (the `pool` branch), signed with
   an automatically rotated GPG signing subkey and deployed to GitHub Pages.
2. **Control plane.** A GitHub App + workflows that onboard other repositories (seed a dispatch credential, install a release-notify
   workflow) so that when a target repo publishes a release, its `.deb` assets are pulled in, re-indexed, re-signed, and republished — with
   zero manual steps.

## Repository layout

| Branch | Purpose                                                                                                                                    |
|--------|--------------------------------------------------------------------------------------------------------------------------------------------|
| `main` | Control-plane + publish workflows, `apt-ftparchive.conf`, docs.                                                                            |
| `pool` | Durable package pool (source of truth) + mirrored public signing cert. **App-only writes** — see [Pool integrity](docs/POOL-INTEGRITY.md). |

The published Pages site (`gh-pages`-style artifact, built by Actions) is generated from `pool` at publish time — it is **not** committed
here.

## Installation

One command drops the signing keyring and an apt source into place:

```sh
curl -fsSL 'https://the-wondersmith.github.io/apt/bootstrap.tgz' \
| sudo tar -C / -xzf - \
&& sudo apt update
```

The bootstrap archive unpacks (rooted at `/`) to:

- `/usr/share/keyrings/wondersmith-apt.gpg` — the dearmored repository signing key
- `/etc/apt/sources.list.d/wondersmith.list` —
  `deb [signed-by=/usr/share/keyrings/wondersmith-apt.gpg] https://the-wondersmith.github.io/apt trixie main`

Prefer to wire it up by hand? The equivalent manual steps:

```sh
curl -fsSL https://the-wondersmith.github.io/apt/pubkey.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/wondersmith-apt.gpg
echo 'deb [signed-by=/usr/share/keyrings/wondersmith-apt.gpg] https://the-wondersmith.github.io/apt trixie main' \
  | sudo tee /etc/apt/sources.list.d/wondersmith.list
sudo apt update
sudo apt install -y wondersmith-apt-keyring  # optional
```

## Signing key

Releases are signed by an offline RSA-4096 certification key (`BE785A845453C6A186AC1EB63FFC3D95F0346B50`) whose short-lived signing subkeys
are rotated automatically. The armored public key is published at the site root (`pubkey.asc`), and the one-line bootstrap install drops its
dearmored form at `/usr/share/keyrings/wondersmith-apt.gpg`.

Because the bootstrap key is a plain file (not managed by `dpkg`), it does **not** update on its own when a signing key is rotated. For
transparent rotation, an **optional** `wondersmith-apt-keyring` package is published into this repository: it ships the same keyring file
under `dpkg` management, so once installed, future key rotations reach the machine via `apt upgrade`.

```sh
sudo apt install wondersmith-apt-keyring
```

This package is entirely optional — the bootstrap install is fully standalone and never depends on it.
