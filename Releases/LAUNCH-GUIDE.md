# LAUNCH-GUIDE — Running WatchNexus Fully & Properly

How to launch WatchNexus v1.0.0 end-to-end — from first boot to a fully
licensed, persistent install. Covers all three tiers (Standard / Pro /
Ultra) and both platforms (Windows / Linux).

> Short version: install the package → open `http://localhost:8001` →
> activate a `WNX-<TIER>-XXXX-XXXX-XXXX` serial → done.
> Everything below is the *proper* path with persistence, service
> supervision, logs, and data in the right place.

---

## 0. What you get

The backend is a **single self-contained binary** (`WatchNexus.Core` /
`WatchNexus.Core.exe`) plus a tier-baked web bundle. The binaries are
identical across tiers — the tier is enforced at runtime by the license
server against the serial you activate.

| Tier | Modules | Serial prefix |
|---|---|---|
| standard | 31 | `WNX-STD-...` |
| pro | 49 | `WNX-PRO-...` |
| ultra | 73 | `WNX-ULT-...` |

Default port: **8001** (override with `WATCHNEXUS_PORT`).

---

## 1. Install

### Windows (recommended: NSIS installer)

```powershell
# Double-click the .exe, or from an admin shell:
.\watchnexus-<tier>-1.0.0-windows-x64.exe /S
```

Installs to `C:\Program Files\WatchNexus`, registers the
`WatchNexusCore` service, adds Start-Menu + Desktop shortcuts.

Data lives in `%PROGRAMDATA%\WatchNexus\` (SQLite DB + logs).

**No installer?** Unzip `program-files/windows-x64/` anywhere and run
`WatchNexus.Core.exe` directly (works, but no service/autostart).

### Linux (DEB / RPM / Arch)

```bash
# Debian / Ubuntu
sudo apt install ./watchnexus-<tier>_1.0.0_amd64.deb

# Fedora / RHEL
sudo dnf install ./watchnexus-<tier>-1.0.0-1.x86_64.rpm

# Arch
sudo pacman -U ./watchnexus-<tier>-1.0.0-1-x86_64.pkg.tar.zst
```

The post-install hook creates the `watchnexus` system user, installs a
systemd unit (`watchnexus.service`), and enables + starts it. Data lives
in `/var/lib/watchnexus/`, logs in `/var/lib/watchnexus/logs/`.

**No package?** Unzip `program-files/linux-x64/` and run
`./WatchNexus.Core` (data then defaults to `./data` next to the binary).

### Docker

```bash
docker run -p 8002:8002 -v watchnexus-data:/data \
  -e WATCHNEXUS_DATA_DIR=/data \
  watchnexus/watchnexus:1.0.0-<tier>
# or docker compose --profile <tier> up -d
```

---

## 2. Verify the service is up

```bash
# Windows
Get-Service WatchNexusCore

# Linux (systemd)
systemctl status watchnexus

# All platforms — the web UI
curl -i http://localhost:8001/api/cellar/first-launch
# -> HTTP/1.1 200 ... (first boot returns first-launch payload)
```

Open `http://localhost:8001` in a browser. You should see the activation
wizard, not an error page.

> Docker note: `docker-compose.yml` maps host `8002 → container 8002`, so
> use `http://localhost:8002` for containers.

---

## 3. Activate a license (required)

Licensing is server-side against `https://licenses.watchnexus.ca`. You need
a valid serial: `WNX-<TIER>-XXXX-XXXX-XXXX` (case-insensitive).

1. On the first-launch screen, enter your serial and the admin email +
   password you want to set.
2. Submit — the server validates the serial by DB lookup, binds it to this
   installation, and returns an activation token.
3. The token is stored locally; `validate` is re-run on app start and every
   few hours. Offline grace is honored (`grace_until`) before enforcement.

Tier is enforced server-side: a `WNX-STD-...` serial unlocks Standard
modules only; entering it on an Ultra-branded build simply exposes the
Standard module set. To move up, activate a higher-tier serial.

### Environment-based seeding (optional, Docker / headless)

```bash
# On first boot these create the admin account without the wizard:
export WATCHNEXUS_SEED_ADMIN_EMAIL=admin@example.com
export WATCHNEXUS_SEED_ADMIN_PASSWORD='a-strong-password'
```

---

## 4. Configuration reference

| Env var | Default | Purpose |
|---|---|---|
| `WATCHNEXUS_PORT` | `8001` | HTTP listen port (`app.Run("http://0.0.0.0:<port>")`) |
| `WATCHNEXUS_DATA_DIR` | Win `%PROGRAMDATA%\WatchNexus`; Linux `/var/lib/watchnexus` (fallback `./data`) | SQLite DB, logs, patches, uploads |
| `JWT_SECRET` | auto-generated & persisted | JWT signing key (rotate = all tokens invalid) |
| `ALLOWED_ORIGINS` | (browser origin) | CORS allowlist, comma-separated |
| `FORCE_HTTPS` | empty | set `1` to redirect HTTP→HTTPS |
| `ASPNETCORE_ENVIRONMENT` | Production (via systemd) | ASP.NET environment |
| `WATCHNEXUS_SEED_ADMIN_EMAIL` / `_PASSWORD` | — | first-boot admin seeding |

---

## 5. Day-to-day operation

### System tray

- **Windows / GUI Linux:** the tray controller starts automatically at
  login (`--tray` mode, `watchnexus-tray.desktop`). Right-click the icon
  for Stop / Restart / Open UI / Logs.
- **Headless Linux:** no tray — use systemd.

### Restart / stop / start

```bash
# Linux (systemd)
sudo systemctl restart watchnexus
sudo systemctl stop watchnexus

# Windows (service)
Restart-Service WatchNexusCore

# Docker
docker compose restart backend   # or: docker restart <container>
```

### Logs

| Platform | Location |
|---|---|
| Windows | `%PROGRAMDATA%\WatchNexus\logs\boot-*.log` (Start Menu → "WatchNexus Logs Folder") |
| Linux (systemd) | `/var/lib/watchnexus/logs/boot-*.log` and `journalctl -u watchnexus` |
| Linux (standalone) | `<install dir>/logs/boot-*.log` |
| Docker | `docker compose logs backend` |

Each boot writes a fresh `boot-<timestamp>.log`. Startup crashes dump the
full exception there.

### Backups

The whole install is a single SQLite file. Backup = copy the DB:

```bash
# Linux
sudo cp /var/lib/watchnexus/watchnexus.db ~/watchnexus-$(date +%F).db

# Windows (admin PowerShell)
Copy-Item "$env:ProgramData\WatchNexus\watchnexus.db" "$HOME\watchnexus-$(Get-Date -f yyyyMMdd).db"
```

> Take the server offline (or accept a few seconds of read-consistency)
> before copying the live `.db`. The built-in Backup UI and WebDAV/Google
> Drive sync automate this — see Settings → Backup.

---

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| Browser shows nothing on `:8001` | Is the service running? `systemctl status watchnexus` / `Get-Service WatchNexusCore`; check `ss -ltnp \| grep 8001` |
| `attempt to write a readonly database` | Data dir not writable. `sudo chown -R watchnexus:watchnexus /var/lib/watchnexus` (Linux) or fix `%PROGRAMDATA%\WatchNexus` ACLs (Windows) |
| First-run screen never activates | Serial typo, or offline at first activation. Check `https://licenses.watchnexus.ca/api/health` from the machine |
| Wrong tier modules shown | The serial tier drives it — activate the correct `WNX-PRO/ULT-...` serial |
| App opens a console and closes | Check the boot log — it pauses with "Press any key" on Windows interactive launches |
| Port already in use | `WATCHNEXUS_PORT=8010 ./WatchNexus.Core` (and open the new port) |

---

## 7. Quick launch matrix

| You have | Command |
|---|---|
| Windows .exe installer | `watchnexus-<tier>-1.0.0-windows-x64.exe` → open `http://localhost:8001` |
| Linux package | `sudo apt/dnf/pacman install ...` → `systemctl status watchnexus` → `http://localhost:8001` |
| Windows program files | `WatchNexus.Core.exe` (from `program-files/windows-x64/`) |
| Linux program files | `./WatchNexus.Core` (from `program-files/linux-x64/`) |
| Docker | `docker run -p 8002:8002 watchnexus/watchnexus:1.0.0-<tier>` → `http://localhost:8002` |
| Electron desktop | packaged app launches backend on port 8001 automatically |

First-boot URL is always the activation wizard; every later boot goes
straight to your library.
