# WatchNexus Releases

Installers for all tiers, per version.

```
Releases/
└── v1.0.0/
    ├── standard/   deb + rpm + pacman + Windows exe + SHA256SUMS
    ├── pro/        same
    └── ultra/      same
```

| Artifact | Install |
|---|---|
| `.deb` | `sudo apt install ./watchnexus-<tier>_1.0.0_amd64.deb` |
| `.rpm` | `sudo dnf install ./watchnexus-<tier>-1.0.0-1.x86_64.rpm` |
| `.pkg.tar.zst` | `sudo pacman -U ./watchnexus-<tier>-1.0.0-1-x86_64.pkg.tar.zst` |
| `.exe` | Double-click (NSIS installer), or `/S` for silent |

After install, open `http://localhost:8001` and activate your
`WNX-<TIER>-XXXX-XXXX-XXXX` serial.

Portable single-file binaries (>100 MB, too large for GitHub) are served
from `releases.watchnexus.ca:/srv/releases/v1.0.0/`. Full upload procedure:
`docs/RELEASE-UPLOAD.md` in the WatchNexus-Master repo.
