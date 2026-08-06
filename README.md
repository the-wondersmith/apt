# pool — durable package pool (source of truth)

This orphan branch is the **authoritative store** of published `.deb` files and
the mirrored public signing certificate. The publish workflow rehydrates this
branch, adds/dedups incoming `.deb` assets, regenerates the signed `apt-ftparchive`
index, deploys it to GitHub Pages, and commits the updated pool back here.

## Layout

```
pool/main/<letter>/<source-package>/<file>.deb   # dedup: same filename overwrites
pubkey.asc                                        # mirrored public signing cert (backup of the 1Password copy)
```

Do not hand-edit. The control-plane workflows own this branch.
