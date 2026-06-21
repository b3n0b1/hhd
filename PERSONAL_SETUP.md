# Personal HHD runtime setup (Konkr Fit on Bazzite)

> Personal branch — **not** for upstream PR. This documents how to get the
> edited HHD build (Konkr Fit support + per-button map) running as a service
> on a fresh Bazzite install, so I don't have to re-figure it out.
>
> Konkr support + button map live on the `konkr-button-profiles` branch. This
> branch (`personal-runtime-docs`) stacks on top of it and just adds this doc.

Context that makes Bazzite annoying:
- It's an immutable OS (bootc/rpm-ostree): `/usr` is read-only, `/etc` is
  writable with sudo.
- SELinux is Enforcing: binaries under `/var/home` are labeled `user_home_t`,
  which systemd refuses to exec (`203/EXEC`) until relabeled `bin_t`.
- Steam Gaming Mode auto-starts the stock `hhd.service`, which grabs the HHD
  lock and shadows the local build unless it's masked.

Paths/names below assume user `benobi` and repo at
`/var/home/benobi/Git/hhd`. Adjust if they change. (`/home` is a symlink to
`/var/home` on Bazzite.)

---

## 1. Clone + check out the working branch

```bash
git clone git@github.com:b3n0b1/hhd.git ~/Git/hhd
cd ~/Git/hhd
git checkout konkr-button-profiles   # or: personal-runtime-docs (includes this doc)
```

## 2. Editable virtualenv

`--system-site-packages` reuses Bazzite's system PyGObject/dbus-python (so we
don't rebuild them, and it works offline). `pip install -e .` makes the `hhd`
binary import from this repo's `src/`, so local edits take effect.

```bash
cd ~/Git/hhd
python -m venv --system-site-packages venv
./venv/bin/pip install -e .
```

Quick sanity check (runs the edited build in the foreground; Ctrl+C to quit):

```bash
sudo systemctl stop hhd@$(whoami) 2>/dev/null   # if the per-user unit is around
sudo ./venv/bin/hhd
```

## 3. SELinux: relabel the venv bin (fixes systemd 203/EXEC)

systemd can't exec anything under `/var/home` until it's labeled `bin_t`:

```bash
sudo chcon -R -u system_u -r object_r --type=bin_t ~/Git/hhd/venv/bin
```

Verify:

```bash
ls -Z ~/Git/hhd/venv/bin/hhd
# want: system_u:object_r:bin_t:s0 ...
```

> **Re-run this any time you recreate the venv or run `pip install -e .`** — it
> can reset the labels and the service will start failing with `203/EXEC`.

## 4. systemd service for the local build

Create `/etc/systemd/system/hhd-local.service` (needs sudo). Exact contents
that work:

```ini
[Unit]
Description=Handheld Daemon Service (local edited build)

[Service]
ExecStart=/var/home/benobi/Git/hhd/venv/bin/hhd
Nice=-12
Restart=on-failure
RestartSec=5

# Mirror stock hhd.service (required for bootc)
SELinuxContext=system_u:unconfined_r:unconfined_t:s0

[Install]
WantedBy=multi-user.target
```

One-liner to write it:

```bash
sudo tee /etc/systemd/system/hhd-local.service >/dev/null <<'EOF'
[Unit]
Description=Handheld Daemon Service (local edited build)

[Service]
ExecStart=/var/home/benobi/Git/hhd/venv/bin/hhd
Nice=-12
Restart=on-failure
RestartSec=5

# Mirror stock hhd.service (required for bootc)
SELinuxContext=system_u:unconfined_r:unconfined_t:s0

[Install]
WantedBy=multi-user.target
EOF
```

## 5. Mask the stock service + enable the local one

Steam Gaming Mode starts `hhd.service` on session start; masking stops it from
ever taking the lock from the local build. Also make sure the per-user unit is
out of the way.

```bash
sudo systemctl disable --now hhd@$(whoami) 2>/dev/null   # if present
sudo systemctl mask --now hhd.service                    # stop + block stock
sudo systemctl daemon-reload
sudo systemctl enable --now hhd-local.service
```

## 6. Verify

```bash
systemctl is-active hhd-local.service     # active
systemctl is-enabled hhd-local.service    # enabled
systemctl is-enabled hhd.service          # masked
ps -eo pid,cmd | grep '[h]hd'             # should be the venv python, not /usr/bin/hhd
```

If the controller section / Konkr Button Map is missing in the HHD overlay,
it's almost always the lock conflict — confirm `hhd.service` is masked and the
running process is `.../venv/bin/python .../venv/bin/hhd`.

---

## Day-to-day

**After editing code in `src/`:** the editable install picks it up; just
restart the service:

```bash
sudo systemctl restart hhd-local.service
```

**After recreating the venv or re-running `pip install -e .`:** redo the
SELinux relabel (step 3), then restart.

**Pull latest of my branch:**

```bash
cd ~/Git/hhd && git pull && sudo systemctl restart hhd-local.service
```

## Restore stock HHD (undo everything)

```bash
sudo systemctl disable --now hhd-local.service
sudo rm /etc/systemd/system/hhd-local.service
sudo systemctl unmask hhd.service
sudo systemctl daemon-reload
sudo systemctl enable --now hhd.service     # or just let Gaming Mode start it
```

---

See `src/hhd/device/ayaneo/KONKR.md` for the controller hardware details
(button layout, evdev codes, the F23 chord, the per-button map).
