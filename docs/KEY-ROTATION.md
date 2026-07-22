# Signing key rotation runbook

This repository signs its APT metadata with **rotating GPG signing subkeys** under a
single offline primary key. Day-to-day rotation is fully automated; this document
covers the one manual task that automation is deliberately barred from doing itself:
**minting a fresh pool of signing subkeys when the current pool runs low.**

## Why this design

- **The primary key never touches CI.** The primary
  (`BE785A845453C6A186AC1EB63FFC3D95F0346B50`) is certify-only and lives offline. CI
  only ever imports a *subkeys-only* bundle with the primary stubbed out. A compromised
  runner therefore leaks only disposable signing subkeys — never the ability to certify
  new keys or revoke the identity.
- **Rotation is time-based and stateless.** The active/next subkey are derived from a
  genesis epoch plus a fixed rotation period, so there is no persisted counter to drift
  or corrupt. [`rotate-keys.yaml`](../.github/workflows/rotate-keys.yaml) computes the
  pair monthly and writes it to secrets; `publish.yaml` only ever *reads* that pair.
- **apt has no "key expiring soon" signal.** For clients, subkey expiry is a silent
  cliff. Two guards front-run it: `rotate-keys.yaml` fails loudly (a failed run emails
  you) well before `publish.yaml`'s hard floor would start refusing to publish.

## When you'll land here

`rotate-keys.yaml` will fail — and email you — in either of these cases:

- **Guard B (pool nearly exhausted):** the active index has reached the last usable pair
  in the pre-staged subkey table (`gpg_subkey_fpr_*`). This is the *early* warning: you
  have roughly one full rotation period (90 days) of lead time. **This is the normal
  trigger for this runbook.**
- **Guard A (newest subkey expiring soon):** the newest subkey in the bundle expires
  inside the `OVERLAP_DAYS + 30` window. This means the whole pool is aging out and must
  be replaced.

Either way the fix is the same: **mint a fresh pool of 8 signing subkeys and re-seed the
repository.**

## Prerequisites

- The **offline primary secret key** on an air-gapped / trusted machine.
- `gpg` (2.2+) and the `op` CLI, authenticated to the vault that holds the signing material.
- Write access to the **GPG item** in secrets (the fresh fingerprints and the subkeys-only bundle are stored there as named fields), plus
  repository admin access to reset the one schedule variable (`SUBKEY_GENESIS_EPOCH`) on `the-wondersmith/apt`.

The current schedule parameters. The first four live in repo **variables**; the two
`OP_*` identifiers live in repo **secrets** (so their values never render in the
Actions UI). Read them before you start:

| Name                   | Kind     | Meaning                                     | Current                                    |
|------------------------|----------|---------------------------------------------|--------------------------------------------|
| `ROTATION_PERIOD_DAYS` | variable | Days each subkey is active                  | `90`                                       |
| `OVERLAP_DAYS`         | variable | Publish-time hard floor / dual-sign overlap | `45`                                       |
| `SUBKEY_GENESIS_EPOCH` | variable | Unix epoch of index 0                       | `1784578890`                               |
| `PRIMARY_FPR`          | variable | Offline primary fingerprint                 | `BE785A845453C6A186AC1EB63FFC3D95F0346B50` |
| `OP_VAULT`             | secret   | vault holding the GPG item                  | `<redacted>`                               |
| `OP_GPG_ITEM`          | secret   | item id for the signing material            | `<redacted>`                               |

The authoritative **subkey fingerprint table** is stored as eight named text fields on the GPG secret item —
`gpg_subkey_fpr_0` through `gpg_subkey_fpr_7`, in rotation order. Both `rotate-keys.yaml`
and `publish.yaml` read the table from those fields by label.

## Step 1 — Mint 8 fresh signing subkeys

On the offline machine, with the primary secret key available. Each subkey is RSA-4096, **sign-only**, with a lifetime one rotation period
longer than the last, so the pool ages out gradually rather than all at once.

```bash
export PRIMARY_FPR='BE785A845453C6A186AC1EB63FFC3D95F0346B50'

# Snapshot the existing subkey fingerprints into a shell variable (not a file) so we
# can subtract them out after minting and never confuse them with the previous pool
# still attached to the primary.
subkeys_before=$(
  gpg --with-colons --list-keys "${PRIMARY_FPR}" \
    | awk -F: '$1 == "fpr" && prev == "sub" { print $10 } { prev = $1 }'
)

# Mint 8 subkeys. Subkey N is valid for (N+1) rotation periods from today, so the
# pool drains one period at a time and the dual-sign overlap always holds.
for idx in $(seq 1 8); do
  gpg --batch --quick-add-key "${PRIMARY_FPR}" rsa4096 sign "$(( idx * 90 ))d"
done
```

> **Why staggered expiry:** if every subkey expired on the same day, Guard A would fire
> for the entire pool at once and there would be no overlap window. Staggering by one
> rotation period keeps exactly one active + one next key valid at every boundary.

## Step 2 — Capture the new fingerprints in index order

The primary still carries the *previous* pool's subkeys, so a plain listing mixes old and
new. Extract **only the keys you just minted**, ordered by creation time — that creation
order *is* the rotation order (index 0 = first minted).

```bash
# List every subkey as "creation-epoch:fingerprint" oldest-first, drop any fingerprint
# that existed before minting (subkeys_before), and print the surviving 8 with their
# index. The creation-epoch sort pins them to rotation order, and the printed index is
# the exact gpg_subkey_fpr_N label you write in Step 5 — no manual counting.
gpg --with-colons --list-keys "${PRIMARY_FPR}" \
  | awk -F: '
      $1 == "sub" { created = $6 }
      $1 == "fpr" && created != "" { print created ":" $10; created = "" }
    ' \
  | sort -t: -k1,1n \
  | grep -vF "${subkeys_before}" \
  | awk -F: '{ printf "gpg_subkeys.gpg_subkey_fpr_%d[text]=%s\n", NR - 1, $2 }'
```

The output is the finished table, ready to transcribe into 1Password. Note the
`gpg_subkeys.` prefix: these fields live in the item's `gpg_subkeys` **section**, so
every `op item edit` assignment must be section-qualified (`section.label[type]=value`).
A bare `gpg_subkey_fpr_0[text]=...` targets a non-existent top-level field and `op`
rejects it with `invalid JSON provided`.

```
gpg_subkeys.gpg_subkey_fpr_0[text]=<fingerprint>
...
gpg_subkeys.gpg_subkey_fpr_7[text]=<fingerprint>
```

You must have exactly 8 lines. If `grep -vF` leaves more or fewer, the before-snapshot
didn't match the primary you minted against — stop and investigate rather than guess.

Verify each is sign-capable and unexpired:

```bash
gpg --with-colons --list-keys "${PRIMARY_FPR}" \
  | awk -F: '$1 == "sub" { print $5, $7, $12 }'   # keyid, expiry-epoch, capabilities
```

## Step 3 — Export the subkeys-only bundle (primary stubbed)

CI must never receive the primary secret. Export the secret **subkeys** while stripping
the primary secret to a stub:

```bash
# Export only the secret subkeys (primary is exported as a stub, not usable to certify).
gpg --batch --pinentry-mode loopback \
    --export-secret-subkeys "${PRIMARY_FPR}" \
  | gpg --enarmor > subkeys-only.asc

# Sanity check: importing this bundle elsewhere must show the primary as "sec#"
# (stubbed / no secret), and the subkeys as "ssb".
```

Confirm the stub before uploading — import into a throwaway `GNUPGHOME` and check the
primary line reads `sec#` (the `#` means "secret not available"):

```bash
verify_home="$(mktemp -d)"
GNUPGHOME="${verify_home}" gpg --import subkeys-only.asc
GNUPGHOME="${verify_home}" gpg --list-secret-keys   # primary must show sec#
```

## Step 4 — Persist the bundle to secrets

Replace the field behind `OP_REF_GPG_SUBKEYS` with the new `subkeys-only.asc` contents:

```bash
op read "op://<vault>/<gpg-item>/<subkeys-field>" >/dev/null   # confirm the ref resolves
op item edit "<gpg-item>" --vault "<vault>" "<subkeys-field>[text]=$(cat subkeys-only.asc)"
```

Leave `OP_REF_GPG_PUBKEY` (the armored public cert) and `OP_REF_GPG_PASSPHRASE`
untouched **unless** the passphrase changed. The public cert already contains the new
subkeys because they were added to the same primary; re-export it too if you want the
served `pubkey.asc` to advertise them immediately:

```bash
gpg --armor --export "${PRIMARY_FPR}" > pubkey.asc
op item edit "<gpg-item>" --vault "<vault>" "<pubkey-field>[text]=$(cat pubkey.asc)"
```

## Step 5 — Update the fingerprint table and reset the cycle

Write the eight new fingerprints into the named secrets fields (rotation order), reset
the rotation-state fields to the sentinel so the next run reseeds cleanly, and reset the
genesis epoch so the new pool starts at index 0:

```bash
# Re-run the Step 2 pipeline and pipe the ready-to-use "key[text]=value" lines straight
# into op — no retyping fingerprints, so they can never be misordered or mistyped.
# (The fingerprints are fixed-length hex with no whitespace, so plain xargs is safe.)
gpg --with-colons --list-keys "${PRIMARY_FPR}" \
  | awk -F: '
      $1 == "sub" { created = $6 }
      $1 == "fpr" && created != "" { print created ":" $10; created = "" }
    ' \
  | sort -t: -k1,1n \
  | grep -vF "${subkeys_before}" \
  | awk -F: '{ printf "gpg_subkeys.gpg_subkey_fpr_%d[text]=%s\n", NR - 1, $2 }' \
  | xargs op item edit "<gpg-item>" --vault "<vault>"

# Clear stale rotation state so publish.yaml won't sign against the old pool before
# the next rotate run reseeds it. rotate-keys.yaml overwrites these on its next run.
# These fields live in the `active_subkeys` section, so each assignment is
# section-qualified (`section.label[type]=value`).
op item edit "<gpg-item>" --vault "<vault>" \
  "active_subkeys.active_fpr[text]=PENDING_ROTATION" \
  "active_subkeys.next_fpr[text]=PENDING_ROTATION" \
  "active_subkeys.fpr_idx[text]=PENDING_ROTATION"

# Reset the genesis epoch to the moment this new cycle should begin. Using "now"
# makes index 0 active immediately.
gh variable set SUBKEY_GENESIS_EPOCH --repo the-wondersmith/apt \
  --body "$(date -u +%s)"
```

> **Why reset the genesis epoch:** the rotation index is
> `(now - SUBKEY_GENESIS_EPOCH) / period`. A fresh pool starts at index 0, so the genesis
> epoch must move to the start of the new cycle or the index will immediately point past
> the start of your new list.

## Step 6 — Re-seed and republish

Run the rotation workflow manually to recompute the pair, write the `active_fpr` /
`next_fpr` / `fpr_idx` fields, and auto-republish:

```bash
gh workflow run rotate-keys.yaml --repo the-wondersmith/apt \
  -f reason="fresh subkey pool minted"
```

Watch it succeed:

```bash
gh run watch --repo the-wondersmith/apt "$(gh run list --repo the-wondersmith/apt \
  --workflow rotate-keys.yaml --limit 1 --json databaseId --jq '.[0].databaseId')"
```

A green run means: guards passed, the `active_fpr` field now holds index 0 of
the new pool (with `next_fpr` = index 1 and `fpr_idx` = 0), and `publish.yaml` has been
dispatched to re-sign the repository.

## Verification checklist

- [ ] `rotate-keys.yaml` run is green (both guards passed).
- [ ] The secrets `active_fpr` / `next_fpr` / `fpr_idx` fields show the new active and
  next fingerprints and index `0`.
- [ ] The dispatched `publish.yaml` run is green.
- [ ] On a clean client, the served metadata verifies and the keyring package advertises
  the new active subkey:

  ```bash
  docker run --rm debian:trixie bash -c '
    apt-get update -qq && apt-get install -y -qq curl ca-certificates gnupg >/dev/null
    curl -fsSL https://the-wondersmith.github.io/apt/bootstrap.tgz | tar -C / -xzf -
    apt-get update
    apt-get install -y wondersmith-apt-keyring
    gpg --show-keys --with-colons /usr/share/keyrings/wondersmith-apt.gpg \
      | awk -F: "\$1==\"fpr\"{print \$10}"
  '
  ```

  The printed fingerprints must include the primary and the freshly minted subkeys.

## What automation still owns (do NOT do these by hand)

- Choosing which subkey is active/next each month — `rotate-keys.yaml` computes it.
- Writing `active_fpr` / `next_fpr` / `fpr_idx` — the workflow owns those fields.
- Kicking off a re-sign after a normal monthly rotation — the workflow auto-dispatches `publish.yaml`.

This runbook is **only** for replenishing the subkey pool. Everything else is hands-off
by design.
