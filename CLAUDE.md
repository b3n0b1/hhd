# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Handheld Daemon (HHD) is a Linux daemon that provides hardware enablement for
Windows gaming handhelds (a vendor-interface replacement à la Armoury Crate):
controller emulation with gyro/back-buttons, TDP/fan control, RGB, and a
gamescope overlay. It runs as a root daemon and talks to evdev/hidraw input
devices, sysfs, and a gamescope overlay UI.

## This fork / branch

This is a personal fork adding **AYANEO Konkr Fit** support. Branch layout:
- `konkr-button-profiles` — Konkr device support + the per-button "Konkr
  Button Map" remap UI (stacked on the original `add-konkr-fit` work).
- `personal-runtime-docs` — stacks the above + `PERSONAL_SETUP.md`; not for PR.

Two docs worth reading before touching Konkr code:
- `src/hhd/device/ayaneo/KONKR.md` — controller hardware: physical button
  layout, captured evdev codes, the F23 chord, and the per-button map design.
- `PERSONAL_SETUP.md` — how to run the edited build as a service on Bazzite.

## Running & developing

There is **no test suite** and no build step (pure Python, editable install).

Bazzite is immutable + SELinux Enforcing, so local edits need an editable venv
(the `local_hhd.sh`/`hhd_cmd.sh` scripts install upstream master from git, so
they do **not** test local edits — don't use them for that):

```bash
python -m venv --system-site-packages venv   # reuse system PyGObject/dbus
./venv/bin/pip install -e .
sudo systemctl stop hhd@$(whoami)             # free the lock
sudo ./venv/bin/hhd                           # run edited build (foreground)
```

- Always use the **absolute venv path**; bare `hhd` runs `/usr/bin/hhd` (the
  packaged build, ignoring local edits).
- `--user <name>` tells the root daemon whose `~/.config/hhd` to use.
- Running it as a systemd service on Bazzite needs an SELinux relabel of the
  venv bin and masking the stock `hhd.service` — see `PERSONAL_SETUP.md`.
- Formatting is **black** (see the badge in `readme.md`); match it.
- After editing `src/`, the editable install picks changes up on restart; no
  reinstall needed.

## Architecture

### Plugin system (entry points → autodetect → lifecycle)
Everything is a plugin discovered via the `hhd.plugins` entry points in
`pyproject.toml`. At startup `src/hhd/__main__.py` iterates them, calling each
`autodetect(existing) -> Sequence[HHDPlugin]`. An autodetect inspects the
hardware (usually DMI `product_name`) and returns plugin instances only if it
matches; returning `[]` means "not this device". Plugins have a `priority`
that orders settings/config application.

`HHDPlugin` (`src/hhd/plugins/plugin.py`) lifecycle:
- `settings()` → returns the config schema (see below).
- `open(emit, context)` → wire up the event emitter and the user context.
- `update(conf)` → called whenever config changes; plugins (re)start their work
  here. Device plugins spawn/replace a worker **thread** from `update()`.
- `close()` → tear down.

The daemon runs as root and uses `--user` + the `Context`
(`switch_priviledge`/`restore_priviledge`) to do per-user config I/O under the
right uid, while keeping device access as root.

### Settings / config schema (YAML-driven)
Plugin settings are declarative YAML loaded with `load_relative_yaml()` next to
the plugin. Node types are defined in `src/hhd/plugins/settings.py`
(`container`, `mode`, `multiple`, `discrete`, `bool`, `int`, `float`,
`display`, ...). Schemas from all plugins are combined with `merge_settings`,
defaults via `parse_defaults`, and user values checked with `validate_config`
(invalid/removed options silently reset to default). YAML anchors/aliases are
supported (loaded with `yaml.safe_load`).

`Config` (`src/hhd/plugins/conf.py`) is the runtime config object. Keys are
dotted paths (`conf["controllers.ayaneo.imu_hz"]`, `conf.get("a.b", default)`);
`to_seq` splits on `.`. A device plugin's worker receives a `Config` already
rooted at its own subtree.

### Controller pipeline (the core)
Defined in `src/hhd/controller/`. Input flows:

```
physical Producers  →  Multiplexer  →  virtual output device
(evdev / hidraw /       (remap, chords,    (DualSense(Edge),
 imu, …)                QAM/guide/overlay,  Xbox, uinput, …)
                        swap_guide)
```

- **Producers/Consumers** (`controller/base.py`): a Producer yields normalized
  `Event`s (`{"type":"button","code":...,"value":...}`, axis, configuration,
  special, rumble). Physical sources live in `controller/physical/`
  (`GenericGamepadEvdev`, `GenericGamepadHidraw`, IMU). Each evdev source has a
  `btn_map: {evdev_code: Button}`; `XBOX_BUTTON_MAP` is the default.
- **Multiplexer** (`controller/base.py`, `Multiplexer.process`) is where almost
  all button logic lives: translating buttons into QAM/guide/overlay actions,
  `swap_guide`, chord/multi-tap detection, the QAM state machine, and emitting
  `special` events to the overlay. It clears a handled event's `code` to `""`
  so it never leaks to the virtual output.
- **Virtual outputs** (`controller/virtual/`): `dualsense`, `uinput`, `sd`
  (Steam Deck), etc. `controller/const.py` defines the canonical `Button`
  vocabulary (`a`,`mode`,`share`,`extra_l1`,`keyboard`,…).

### Device plugins
`src/hhd/device/<vendor>/`. Pattern: `__init__.py` holds the plugin +
`autodetect` (DMI lookup against a `CONFS` dict in `const.py`); `*.yml` are the
settings schema; `base.py`'s `plugin_run` builds the Producer→Multiplexer→
virtual pipeline for that device using a per-device `dconf` dict (flags like
`extra_buttons`, `rgb`, and Konkr's `mode_is_guide`/`face_remap`). Device
quirks are expressed as `dconf` flags read in `base.py`, **gated** so they
don't affect other devices sharing the module.

### Overlay & special events
`src/hhd/plugins/overlay/` renders the gamescope UI (QAM side menu + expanded
view). The Multiplexer/emit sends `{"type":"special","event":...}`; the overlay
maps these to commands (`overlay`/`qam_double`→`open_qam`,
`qam_triple`→`open_expanded`, `guide`→Steam). User-facing shortcut action
names/values are centralized in `overlay/shortcuts.yml` (the `&shortcuts`
anchor: `hhd_qam`/"HHD Side Menu", `hhd_expanded`/"HHD Overlay",
`steam_qam`/"Steam Side Menu", …). **Reuse these exact names** when adding any
mapping UI — don't invent labels.

### adjustor
`src/adjustor/` is a separate plugin package (own entry point) for TDP/power
management (per-vendor drivers under `adjustor/drivers/`, a FUSE shim under
`adjustor/fuse/`). Mostly independent of the controller pipeline.

## Adding device support (quick recipe)
1. Find the vendor module under `src/hhd/device/` whose controller hardware
   matches (same VID/PIDs).
2. Add a `CONFS["<DMI PRODUCT NAME>"]` entry in that module's `const.py` with
   the appropriate flags.
3. If the device needs different button handling, add a `dconf` flag and branch
   on it in `base.py` — keep it gated so existing devices are untouched.
4. Test with the editable venv flow above; there are no automated tests, so
   verify on-device.
