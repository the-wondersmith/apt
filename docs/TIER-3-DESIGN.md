# Tier-3 design: Contents indices + multi-suite

Status: **design captured, not yet implemented.** Picking this up after a power outage.

This document is the authoritative record of the Tier-3 plan and the decisions
made so far. Tier-1 (arch-floor guard, per-arch Packages split, CI lint, op-CLI
pin) and Tier-2 (provenance-verified ingest, signed pool manifest, signed
per-run provenance log, pool-branch App-only ruleset, autonomous Dependabot) are
already complete, live, and validated end-to-end. Tier-3 is the tail of the
"hardening / monitoring / scale" phase and covers the two remaining deferred
items:

- **#6** — per-arch signed `Contents` indices (`apt-file` support)
- **#5** — multi-suite support

---

## Decisions locked so far

1. **Contents scope** — *Per-arch Contents, signed.* Generate `Contents-all` and
   `Contents-<arch>` with per-arch filtering that matches the Packages split,
   gzip them, and place them so the `Release` signature covers them.
2. **Sequencing** — *Design both together now* (multi-suite touches pool layout,
   so the layout is decided once, up front) before writing any code.

Two decisions still **open** (to answer when we resume):

- **#6:** `.gz`-only vs. keep the plain `Contents-<arch>` file too.
- **#5:** build Option A now (with a one-time pool migration) vs. lock the
  Option-A design as the plan and defer implementation until a second suite
  actually exists.

**Recommendations:** for #6, ship `.gz` only (Debian practice; apt clients only
fetch the `.gz`). For #5, lock Option A as the design but **defer
implementation** until a second Debian suite genuinely matters — everything today
is `Architecture: all` on a single `trixie`, so building the suite-loop + pool
migration now adds real complexity and migration risk for zero current benefit.

---

## Verified mechanics (docker debian:trixie, apt 3.0.3)

- `apt-ftparchive contents <dir>` walks a directory **recursively**, emits
  `path<TAB>package` lines, and does **no** architecture filtering (same
  limitation as `apt-ftparchive packages`).
- Contents output is **not** stanza-based (unlike `Packages`), so the
  awk-stanza split trick used for the Packages per-arch split does **not** apply
  to Contents. Per-arch Contents must instead be produced by feeding
  `apt-ftparchive contents` a directory that already contains only the relevant
  `.deb` files.
- `apt-ftparchive ... release dists/<suite>` hashes **every file it finds** under
  that tree, so any `Contents-*` placed under `dists/<suite>/...` is
  automatically listed in `Release` and therefore covered by `InRelease` +
  `Release.gpg`.

---

## #6 — Per-arch signed `Contents` indices (ready to write)

**Value:** enables `apt-file search /path/to/file` → package lookups for clients.

**Where:** inside the existing *Build indices, Release, and dual-sign* step in
`.github/workflows/publish.yaml`, **after** the Packages per-arch loop and
**before** the `apt-ftparchive ... release` call, so the Release checksums cover
the new files and the existing dual-sign step signs them for free.

**Mechanism** (mirrors the Packages arch-split semantics exactly — `all` appears
in every arch's Contents; a real-arch build appears only in its own):

```bash
# after the binary-<arch>/Packages loop, still in `site/`
for arch in all $ARCHES; do
  staging_dir="$(mktemp -d)"
  for deb in pool/**/*.deb; do
    deb_arch="$(dpkg-deb -f "${deb}" Architecture)"
    if [ "${deb_arch}" = "all" ] || [ "${deb_arch}" = "${arch}" ]; then
      ln -s "$(readlink -f "${deb}")" "${staging_dir}/$(basename "${deb}")"
    fi
  done
  apt-ftparchive contents "${staging_dir}" \
    > "dists/${SUITE}/${COMPONENT}/Contents-${arch}"
  gzip -kf "dists/${SUITE}/${COMPONENT}/Contents-${arch}"   # -f only if .gz-only
  rm -rf "${staging_dir}"
done
```

Notes:
- `ARCHES='amd64 arm64'`, so the loop produces `Contents-all`, `Contents-amd64`,
  `Contents-arm64`.
- Requires `globstar` for `pool/**/*.deb` (bash `shopt -s globstar`) **or** use
  `find pool -name '*.deb'` to avoid relying on it — prefer `find` for
  portability, matching house style.
- **Placement:** `dists/<suite>/<component>/Contents-<arch>.gz` (component-scoped,
  matching where `binary-<arch>/` lives). Debian also allows suite-root
  placement; component-scoped is correct for a componentized repo and stays under
  the `apt-ftparchive release` walk.
- **Signature coverage:** these files live under `dists/<suite>/...` inside
  `site/`, so `Release` lists them and the dual-sign step signs them. **No
  `pool.sha256` manifest change** — Contents are Pages build artifacts under
  `site/`, not pool source, exactly like `Packages`/`Release` today. They are
  **not** persisted to the pool branch.

**Open sub-decision:** `.gz`-only (`gzip -f`, drop the plain file) vs. keep both
(`gzip -kf`). apt clients fetch only `Contents-<arch>.gz`. Debian ships `.gz`
only. Recommended: `.gz` only.

---

## #5 — Multi-suite (design locked; implementation TBD)

**Core problem:** a `.deb` has **no concept of suite**. Suite (`trixie`,
`bookworm`, …) is purely a repo-layout dimension, so multi-suite is fundamentally
a decision about *where suite enters the system*.

### Option A — Suite in the dispatch payload + suite in the pool path (RECOMMENDED)

- `client_payload.suite` (already plumbed into `env.SUITE`, currently unused for
  routing) becomes the routing key.
- Pool layout gains a suite level: `pool/<suite>/<component>/<letter>/<pkg>/*.deb`
  (today it is `pool/<component>/<letter>/<pkg>/*.deb`).
- Ingest pools into the dispatched suite; the Release/dual-sign logic loops over
  the suites that exist under `pool/`.
- Bootstrap `.list` gains one `deb ... <suite> <component>` line per suite (or
  stays single-suite for the default and documents the rest).
- **Cleanest, most standard**, but touches pool layout → requires a **one-time
  migration** of the existing flat pool: move `pool/main/**` →
  `pool/trixie/main/**`, then re-sign `pool.sha256`.

### Option B — Flat pool, single active suite, reconfigurable

- Keep `pool/<component>/…` flat; each run targets exactly one suite and rebuilds
  only that suite's `dists/<suite>/`.
- Multiple suites only coexist if their pools are disjoint — which they are not
  here (the same `.deb` serves all). Effectively "single active suite,
  reconfigurable," **not true multi-suite.** Lowest effort, weakest capability.

### Option C — Per-suite pool branches (`pool-trixie`, `pool-bookworm`)

- Strong isolation, but multiplies the ruleset / branch-protection / manifest
  machinery per suite. Overkill for a personal fleet.

### Recommendation

Lock **Option A** as the design. **Defer implementation** until a second suite
genuinely exists. Rationale: today everything is `Architecture: all` on a single
`trixie`; building the suite-loop + pool migration now adds complexity and
migration risk for zero present benefit.

### Option A implementation checklist (for when we build it)

1. **Pool migration (one-time, USER-run git ops):** move `pool/main/**` →
   `pool/trixie/main/**` on the `pool` branch; regenerate + re-sign `pool.sha256`
   / `pool.sha256.asc`; commit + push as the App (pool is App-only via ruleset
   `pool-app-only` id 19575089).
2. **Ingest** (`publish.yaml`): pool into
   `site/pool/${SUITE}/${COMPONENT}/${pool_letter}/${package_name}` (add `SUITE`
   level). `SUITE` comes from `client_payload.suite` (default `trixie`).
3. **Rehydrate / Verify integrity / Persist / manifest `find`:** update the pool
   `find` paths to include the suite level; the reverse-check and manifest
   regeneration walk `pool provenance` — those stay correct as long as the new
   suite dirs are under `pool/`.
4. **Build indices step:** wrap the existing Packages + Contents + Release +
   dual-sign logic in a `for suite in <suites present under pool/>` loop, each
   building `dists/<suite>/...` and signing `dists/<suite>/{InRelease,Release.gpg}`.
5. **Bootstrap tarball / README:** parameterize the `wondersmith.list` `deb`
   line(s) by suite; update the README distribution table + manual-install
   example (currently hardwires `trixie`).
6. **arch-floor guard:** unaffected structurally (still per-package: `all` OR both
   `amd64`+`arm64`).

---

## Exact edit sites in `publish.yaml` (current state: 602 lines)

Key regions (line numbers approximate, re-verify on resume):

- **Build indices, Release, and dual-sign** — approx. lines 420–480. Contains
  `apt-ftparchive packages pool > /tmp/Packages.full` (~431), the
  `for arch in all $ARCHES` awk-split into
  `dists/${SUITE}/${COMPONENT}/binary-${arch}/Packages` (~432–449), the
  `apt-ftparchive ... release dists/${SUITE} > dists/${SUITE}/Release` (~452–458),
  and the dual-sign (`gpg --clearsign ... -o InRelease Release` ~463–471;
  `gpg -abs -o Release.gpg Release` ~473–480).
  **#6 Contents generation goes here, after the Packages loop and before the
  release call.**
- **Build bootstrap tarball** — approx. lines 490–511. Writes
  `root/etc/apt/sources.list.d/wondersmith.list` =
  `deb [signed-by=/usr/share/keyrings/wondersmith-apt.gpg] https://the-wondersmith.github.io/apt ${SUITE} ${COMPONENT}`.
  **#5 multi-suite touches this** (single suite hardwired).
- **Ingest** — approx. lines 307–406. Pool loop pools into
  `site/pool/${COMPONENT}/${pool_letter}/${package_name}`.
  **#5 adds the `${SUITE}` level here.**
- **Persist pool** — approx. lines 553–593. `find pool provenance -type f
  ! -name .gitkeep | ... sha256sum > pool.sha256` (~575) + detach-sign.
  **#5 updates pool paths here** (still works as long as suite dirs live under
  `pool/`).

`env.SUITE` default `trixie`, `env.ARCHES` = `amd64 arm64`, `env.COMPONENT` =
`main`, each resolved from workflow inputs || `client_payload` || literal
(~lines 56–59).

`apt-ftparchive.conf` is static Release identity only (Origin/Label
`Wondersmith APT`); Suite/Codename/Components/Architectures are passed via runtime
`-o` overrides — no change needed for #6; #5 needs no structural change there
either (the per-suite loop just re-invokes with different `-o Suite=`).

---

## Validation plan (both items)

- `actionlint /Users/wondersmith/.golang/bin/actionlint .github/workflows/publish.yaml`
- `yamllint --strict -c .yamllint.yaml .`  (validate against the **repo's own**
  config — the machine has a personal `~/.config/yamllint/config` that relaxes
  rules and gives false-green with bare `yamllint --strict .`).
- Local dry-run in docker debian:trixie: build a throwaway pool with an
  `Architecture: all` .deb (and, if testing multi-arch, synthetic amd64/arm64
  stubs), run the Contents loop, confirm `Contents-all` / `Contents-<arch>`
  contents are correctly filtered, and confirm `apt-ftparchive release` lists the
  `Contents-*.gz` in the generated `Release`.
- End-to-end: dispatch `publish.yaml` (workflow_dispatch re-sign, or a real
  pve-modkit release), confirm signed `Contents-*.gz` served, and
  `apt-get install apt-file && apt-file update && apt-file search <path>` resolves
  against our repo on `docker.io/dockurr/proxmox:9.2` (run with
  `--privileged --entrypoint bash`).

---

## Standing constraints (unchanged)

- **USER** does all git commit/push/tag/release + secret/variable/1Password/infra
  /rulesets ops + repo-settings toggles. The assistant only edits the working
  tree, runs read-only/verify bash, and dispatches workflows; it hands over exact
  commands and never runs write ops. Each diff is shown for approval before
  writing in batch work.
- Workflow files are `.yaml`, never `.yml`; `publish.yaml` is edited in place.
- Descriptive variable names (no opaque single letters; awk loop var `line` is
  fine).
- `ast_grep_replace` may report APPLIED but not persist — re-verify with grep;
  prefer python/perl in-place for critical edits.

---

## Resume checklist (start here when power is back)

1. Re-read this document.
2. Answer the two open decisions (#6 `.gz`-only?; #5 build-now vs. defer).
3. If #6 approved: write the Contents loop into the Build-indices step, lint,
   dry-run in docker, then hand USER the commit command.
4. If #5 approved for now: follow the Option A implementation checklist above
   (pool migration is a USER git op); otherwise mark #5 as "designed, deferred"
   and the Tier-3 / hardening arc is complete.
