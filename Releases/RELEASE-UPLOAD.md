# RELEASE-UPLOAD — Publishing Installers to the WN-Admin/WatchNexus Repo

How to upload the WatchNexus v1.0.0 installers to the
[WN-Admin/WatchNexus](https://github.com/WN-Admin/WatchNexus) repository,
which hosts releases for the update system and for manual download.

> This repo has **three top-level folders**:
>
> | Folder | Purpose |
> |---|---|
> | `Releases/` | Full installer packages (deb / rpm / pacman / exe) per version per tier — **upload here** |
> | `Updates/` | Incremental patch payloads consumed by the built-in updater |
> | `Patches/` | Source of the patch manifests / apply scripts |

---

## 1. What goes where

### Releases (this doc)

Per-version per-tier installer artifacts, all under 100 MB so they fit
GitHub's per-file limit:

```
Releases/
└── v1.0.0/
    ├── standard/
    │   ├── watchnexus-standard_1.0.0_amd64.deb
    │   ├── watchnexus-standard-1.0.0-1.x86_64.rpm
    │   ├── watchnexus-standard-1.0.0-1-x86_64.pkg.tar.zst
    │   ├── watchnexus-standard-1.0.0-windows-x64.exe
    │   └── SHA256SUMS.txt
    ├── pro/       (same 5 files)
    └── ultra/     (same 5 files)
```

> The single-file backend binaries (`WatchNexus.Core.exe` ~154 MB,
> `WatchNexus.Core` ~113 MB) **exceed GitHub's 100 MB limit** — they are
> served from the release webserver instead:
> `releases.watchnexus.ca:/srv/releases/v1.0.0/` (see `WN_Releases/README.md`).

### Updates / Patches

- `Updates/` is populated by the update tooling (`docs/UPDATE-SYSTEM.md`)
  with binary patch payloads.
- `Patches/` holds the manifests those updates reference.

---

## 2. Generate the artifacts

From `WatchNexus-Master` (requires fish, fpm, makensis, rpmbuild — see
`docs/BUILD-INSTALLERS.md`):

```bash
./build/build-installers.fish all          # deb/rpm/pacman/exe for all tiers
```

Output lands in `release/<tier>/`. The `SHA256SUMS.txt` files are produced
automatically (Step 7 of the script).

---

## 3. Copy into the releases repo

```bash
# From a fresh clone:
git clone https://github.com/WN-Admin/WatchNexus.git
cd WatchNexus

mkdir -p Releases/v1.0.0/{standard,pro,ultra}

for tier in standard pro ultra; do
  cp /path/to/WatchNexus-Master/release/$tier/deb/*.deb       Releases/v1.0.0/$tier/
  cp /path/to/WatchNexus-Master/release/$tier/rpm/*.rpm       Releases/v1.0.0/$tier/
  cp /path/to/WatchNexus-Master/release/$tier/arch/*.pkg.tar.zst Releases/v1.0.0/$tier/
  cp /path/to/WatchNexus-Master/release/$tier/windows/*.exe   Releases/v1.0.0/$tier/
  cp /path/to/WatchNexus-Master/release/$tier/SHA256SUMS.txt  Releases/v1.0.0/$tier/
done
```

---

## 4. Commit + push

```bash
cd WatchNexus
git add Releases/
git commit -m "release: v1.0.0 installers (standard/pro/ultra, windows+linux)"
git push origin main
```

> GitHub warning: files over 100 MB are rejected by the API/UI. The
> installers are 38–54 MB each, so this is safe. If you ever need to
> remove a large file from history, use `git filter-repo` before pushing.

---

## 5. Verify after push

1. `https://github.com/WN-Admin/WatchNexus/tree/main/Releases/v1.0.0` shows
   all 12 installer files + 3 `SHA256SUMS.txt`.
2. Cross-check a checksum:
   ```bash
   curl -sL https://github.com/WN-Admin/WatchNexus/raw/main/Releases/v1.0.0/ultra/SHA256SUMS.txt
   sha256sum WatchNexus-Master/release/ultra/windows/*.exe
   ```

---

## 6. Daily-pattern release (update the "latest" pointer)

Repos don't hold symlinks, so keep versioned folders and point your
download page / update manifests at `v1.0.0`. When v1.1.0 ships, add
`Releases/v1.1.0/` and leave older versions intact for downgrade support.

---

## 7. Release checklist

- [ ] All 4 installer formats + SHA256SUMS exist per tier.
- [ ] No file exceeds 100 MB (installers are all < 60 MB).
- [ ] `SHA256SUMS.txt` hashes match the committed artifacts.
- [ ] Committed to `Releases/v<version>/<tier>/` and pushed.
- [ ] Webserver copy mirrored at `releases.watchnexus.ca` (for binaries >
      100 MB and the portable program files).
- [ ] Update system (`Updates/`) pointed at the new version, if shipping
      incremental patches.
