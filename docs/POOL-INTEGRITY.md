# Pool integrity & break-glass

The `pool` branch is the **durable source of truth** for every package this
repository serves. `publish.yaml` re-indexes and re-signs whatever is on `pool`,
so anything that lands there is trusted input to the signing step. This document
describes how `pool` is protected, why the protection is layered the way it is,
and how to regain write access in an emergency.

## Two layers of protection

Integrity rests on a cryptographic layer (the real guarantor) backed by an
access-control layer (defense-in-depth). Either alone is weaker than both
together.

### 1. Cryptographic chain (the real guarantor)

`publish.yaml` treats `pool` as untrusted-until-verified. On every run it proves
the branch is internally consistent **before** it signs anything, and it leaves
behind fresh proofs for the next run:

- **Signed manifest — `pool.sha256` + `pool.sha256.asc`.** A `sha256sum` listing
  of every pooled file plus the provenance log, detached-signed with the active
  signing subkey. Before ingesting, publish runs `gpg --verify` on the signature,
  `sha256sum --strict -c` on the digests, and a reverse-check that no file exists
  outside the manifest. Any tampered, added, or removed file fails the run loudly.
- **Per-run provenance log — `provenance/<run-id>.txt` + `.asc`.** An append-only,
  detached-signed record of each publish: who dispatched it, the source repo/tag,
  the ingested asset filenames + SHA-256s, the active fingerprint, and a
  timestamp. Run-keyed, so history is never overwritten.

Because these are signed by a subkey whose secret half only ever exists inside an
ephemeral CI keyring gated by the 1Password passphrase, an attacker who somehow
writes to `pool` cannot forge a matching signature. **This is why the manifest is
signed and not merely committed:** an unsigned checksum file is only as trustworthy
as write access to the branch — an attacker who can inject a file could also
rewrite the checksum to cover it.

### 2. App-only push restriction (defense-in-depth)

`pool` is push-restricted to a single identity: the **`wondersmith-apt-control-plane`
GitHub App** (App ID `4349264`). No human — including the repository owner — can
push, force-push, update, or delete `pool` interactively. This is enforced with a
repository **ruleset** named `pool-app-only`:

| Field           | Value                                                  |
|-----------------|--------------------------------------------------------|
| `target`        | `branch`                                               |
| `enforcement`   | `active`                                               |
| ref condition   | `refs/heads/pool`                                      |
| rules           | `update`, `deletion`, `non_fast_forward`               |
| `bypass_actors` | Integration `4349264` (`wondersmith-apt-control-plane`), `bypass_mode: always` |

> **Why a ruleset and not branch protection?** Branch-protection `restrictions`
> (user/team/app allow-lists) are an **organization-only** feature. `the-wondersmith`
> is a personal account, so that field returns
> `422 Only organization repositories can have users and team restrictions`.
> Repository **rulesets** support per-actor `bypass_actors` on user-owned repos,
> which is how a single-App push restriction is expressed here: the rules block
> everyone, and only the App is in the bypass list.

The ruleset lives in GitHub's settings, not in this tree. Recreate it with:

```bash
cat > /tmp/pool-app-only.json <<'JSON'
{
  "name": "pool-app-only",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [
    { "actor_id": 4349264, "actor_type": "Integration", "bypass_mode": "always" }
  ],
  "conditions": { "ref_name": { "include": ["refs/heads/pool"], "exclude": [] } },
  "rules": [ { "type": "update" }, { "type": "deletion" }, { "type": "non_fast_forward" } ]
}
JSON

gh api -X POST repos/the-wondersmith/apt/rulesets --input /tmp/pool-app-only.json
rm /tmp/pool-app-only.json
```

Confirm it is active and correctly scoped:

```bash
ruleset_id="$(gh api repos/the-wondersmith/apt/rulesets \
  --jq '.[] | select(.name == "pool-app-only") | .id')"

gh api "repos/the-wondersmith/apt/rulesets/${ruleset_id}" --jq '{
  enforcement,
  bypass: [.bypass_actors[].actor_id],
  rules:  [.rules[].type],
  refs:   .conditions.ref_name.include
}'
# expect: enforcement "active", bypass [4349264],
# rules ["update","deletion","non_fast_forward"], refs ["refs/heads/pool"]
```

> **Do not add classic branch protection to `pool`.** The `pool-app-only` ruleset
> is the sole intended mechanism. Classic branch protection cannot express the
> single-App push restriction on a user-owned repo (see above) and would only
> duplicate — and muddy — the rules the ruleset already enforces. If you ever find
> a classic protection on `pool` (e.g. left over from an earlier experiment),
> remove it with `gh api -X DELETE repos/the-wondersmith/apt/branches/pool/protection`
> and confirm `pool-app-only` still stands.

## `main` branch protection

Where `pool` is machine-only, `main` is human-only: it holds the workflows,
actions, and docs that define the control plane. It is protected by a separate,
**unbypassable** ruleset named `main-protected` that requires every change to
land through a reviewed pull request. Nobody — not the App, not the repository
owner — can push directly to `main` or force-push/delete it while the ruleset is
active.

| Field           | Value                                            |
|-----------------|--------------------------------------------------|
| `target`        | `branch`                                         |
| `enforcement`   | `disabled` (inert until you turn it on)          |
| ref condition   | `refs/heads/main`                                |
| rules           | `pull_request`, `deletion`, `non_fast_forward`   |
| `bypass_actors` | *(empty — truly unbypassable)*                   |

The `pull_request` rule requires 1 approving review, dismisses stale approvals on
new pushes, requires the last push to be approved by someone other than its
author, and requires all review threads resolved before merge.

> **It ships disabled on purpose.** Turning it on means *you* also need a PR to
> change `main` — no more direct pushes. Enable it once you're ready to work that
> way (see below). Until then it exists but enforces nothing.

Create it (starts disabled):

```bash
cat > /tmp/main-protected.json <<'JSON'
{
  "name": "main-protected",
  "target": "branch",
  "enforcement": "disabled",
  "bypass_actors": [],
  "conditions": { "ref_name": { "include": ["refs/heads/main"], "exclude": [] } },
  "rules": [
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": true,
        "required_review_thread_resolution": true,
        "automatic_copilot_code_review_enabled": false,
        "allowed_merge_methods": ["merge", "squash", "rebase"]
      }
    },
    { "type": "deletion" },
    { "type": "non_fast_forward" }
  ]
}
JSON

gh api -X POST repos/the-wondersmith/apt/rulesets --input /tmp/main-protected.json
rm /tmp/main-protected.json
```

Confirm it exists, disabled and correctly scoped:

```bash
main_ruleset_id="$(gh api repos/the-wondersmith/apt/rulesets \
  --jq '.[] | select(.name == "main-protected") | .id')"

gh api "repos/the-wondersmith/apt/rulesets/${main_ruleset_id}" --jq '{
  enforcement,
  bypass: [.bypass_actors[].actor_id],
  rules:  [.rules[].type],
  refs:   .conditions.ref_name.include
}'
# expect: enforcement "disabled", bypass [],
# rules ["pull_request","deletion","non_fast_forward"], refs ["refs/heads/main"]
```

Turn it on when you're ready to adopt PR-only changes to `main`:

```bash
gh api -X PUT "repos/the-wondersmith/apt/rulesets/${main_ruleset_id}" \
  -f enforcement=active -f name=main-protected
```

## Break-glass: regaining write access to `pool`

You should almost never need this. Legitimate changes to `pool` flow through
`publish.yaml` (which pushes as the App). Reach for break-glass only to repair a
genuinely stuck pool — e.g. a corrupted manifest that blocks every publish, or a
bad commit that must be surgically removed.

The cryptographic chain keeps you honest while the gate is open: after any manual
edit you **must** regenerate and re-sign `pool.sha256`, or the next `publish.yaml`
run will fail its integrity check. Prefer fixing forward through the workflow over
hand-editing whenever possible.

### Step 1 — Open the gate

Two options, least-invasive first.

**Option A (recommended): disable the ruleset temporarily.**

```bash
ruleset_id="$(gh api repos/the-wondersmith/apt/rulesets \
  --jq '.[] | select(.name == "pool-app-only") | .id')"

gh api -X PUT "repos/the-wondersmith/apt/rulesets/${ruleset_id}" \
  -f enforcement=disabled -f name=pool-app-only
```

**Option B: add yourself to the bypass list.** Fetch your user actor id, append a
`bypass_actors` entry (`actor_type: "OrganizationAdmin"` is not valid on a user
repo — use your own `actor_type: "Integration"`/user id via the rulesets UI if
Option A is unavailable). Option A is simpler and fully reversible, so prefer it.

### Step 2 — Make the repair

```bash
git fetch origin pool
git switch pool     # or: git worktree add ../apt-pool pool
# ... surgical fix ...
```

### Step 3 — Re-establish the cryptographic proof

Any manual change invalidates the signed manifest. Regenerate and re-sign it with
the **active** signing subkey (the same key `publish.yaml` uses). You need the
subkeys-only bundle imported into a scratch `GNUPGHOME` and the signing
passphrase — see [`KEY-ROTATION.md`](KEY-ROTATION.md) for how the bundle is
structured.

```bash
# From the pool worktree root, with the active subkey available and $ACTIVE_FPR set:
( find pool provenance -type f ! -name .gitkeep | LC_ALL=C sort | xargs -r sha256sum ) \
  > pool.sha256

gpg --yes --batch --pinentry-mode loopback \
    -u "${ACTIVE_FPR}!" --detach-sign --armor \
    -o pool.sha256.asc pool.sha256

git add -A
git commit -m "pool: manual repair (break-glass) + re-sign manifest"
git push origin HEAD:pool
```

> The `find` deliberately excludes `pool.sha256` / `pool.sha256.asc` (they sit at
> the branch root, outside `pool/` and `provenance/`) so the manifest never tries
> to hash its own signature.

### Step 4 — Close the gate immediately

Re-enable enforcement (Option A) or remove your bypass entry (Option B):

```bash
gh api -X PUT "repos/the-wondersmith/apt/rulesets/${ruleset_id}" \
  -f enforcement=active -f name=pool-app-only
```

Confirm with the verification `gh api ... --jq` block above (`enforcement` back to
`"active"`, `bypass` back to `[4349264]`).

### Step 5 — Verify the chain end-to-end

Dispatch a re-sign and confirm publish's integrity check passes against your
hand-signed manifest:

```bash
gh workflow run publish.yaml --repo the-wondersmith/apt \
  -f reason="post break-glass re-sign"
```

A green run means the manifest you re-signed verified cleanly and the repository
was re-indexed and re-signed. If it fails at "Verify durable pool integrity", the
manifest and the pool contents disagree — recheck Step 3.
