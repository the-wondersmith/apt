# apt — Wondersmith APT package repository & control plane

A GPG-signed [Debian APT repository](https://the-wondersmith.github.io/apt) served
via GitHub Pages, plus the **control plane** that onboards and feeds package repos
into it.

- **Distribution:** `trixie` (Debian 13 / Proxmox VE 9) · **Component:** `main` · **Arch:** `all`
- **Install:** see [Installation](#installation) below.

This repository does two jobs:

1. **APT repository host.** A stateless, `apt-ftparchive`-generated index over a
   durable package **pool** (the `pool` branch), signed with an automatically
   rotated GPG signing subkey and deployed to GitHub Pages.
2. **Control plane.** A GitHub App + workflows that onboard other repositories
   (seed a dispatch credential, plant a release-notify workflow) so that when a
   target repo publishes a release, its `.deb` assets are pulled in, re-indexed,
   re-signed, and republished — with zero manual steps.

## Repository layout

| Branch | Purpose |
| ------ | ------- |
| `main` | Control-plane + publish workflows, `apt-ftparchive.conf`, docs. |
| `pool` | Durable package pool (source of truth) + mirrored public signing cert. |

The published Pages site (`gh-pages`-style artifact, built by Actions) is generated
from `pool` at publish time — it is **not** committed here.

## Installation

> Bootstrapping in progress — install instructions will be finalized once the
> day-zero publish lands.

## Signing key

Releases are signed by an offline RSA-4096 certification key
(`BE785A845453C6A186AC1EB63FFC3D95F0346B50`) whose short-lived signing subkeys are
rotated automatically. The armored public key is published at the site root and
shipped in an archive-keyring package so key rotations reach clients transparently
via `apt upgrade`.
