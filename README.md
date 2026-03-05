# omabright

Per-monitor brightness control for Omarchy/Hyprland.
Brightness keys always target the monitor under your cursor.

## What It Does

- Cursor-aware brightness routing across multiple monitors
- Native laptop backlight control via `brightnessctl`
- External monitor hardware brightness via DDC/CI (`ddcutil`)
- Automatic SDR fallback (`sdrBrightness`) when DDC is unavailable
- Fast daemon + socket IPC for low-latency key-repeat handling
- OSD feedback via `swayosd-client` (default) or `notify-send`

## Runtime Requirements

`omabright` must run inside an active Hyprland graphical session.
Running in headless or non-Hyprland environments is not supported for functional validation.

Core dependencies:
- `hyprctl`
- `brightnessctl`
- `systemd --user`

Optional but recommended for external monitors:
- `ddcutil`
- Access to `/dev/i2c-*` (via `uaccess` or `i2c` group)

Optional OSD:
- `swayosd-client`

## Agent-First Install (Recommended)

Run from a repository checkout:

```bash
REPO_DIR="/path/to/omarchy_display_brightness"
cd "$REPO_DIR"
```

Preflight checks (fail early):

```bash
command -v hyprctl brightnessctl systemctl python3 >/dev/null
test -n "${XDG_RUNTIME_DIR:-}"
test -n "${WAYLAND_DISPLAY:-}"
```

If `ddcutil` is installed, external-monitor hardware brightness can be used.
Without `ddcutil`, external monitors use SDR fallback.

Install and start:

```bash
./bin/omabright install
systemctl --user daemon-reload
systemctl --user enable --now omabright.service
```

Notes:
- `./bin/omabright install` installs the executable to `~/.local/bin/omabright`
- It also installs the service file to `~/.config/systemd/user/omabright.service`
- Service is attached to `graphical-session.target` to avoid early-start race issues

## Agent Verification (Machine-Checkable)

A successful install should satisfy all checks below:

```bash
systemctl --user is-active omabright.service
~/.local/bin/omabright ping
~/.local/bin/omabright status
~/.local/bin/omabright list
```

Strict pass/fail assertions:

```bash
test "$(systemctl --user is-active omabright.service)" = "active"
~/.local/bin/omabright ping | python3 -c 'import json,sys; d=json.load(sys.stdin); assert d.get("ok") is True'
~/.local/bin/omabright status | python3 -c 'import json,sys; d=json.load(sys.stdin); assert d.get("ok") is True; assert d.get("backend") in {"backlight","ddc","sdr"}'
~/.local/bin/omabright list | python3 -c 'import json,sys; d=json.load(sys.stdin); assert d.get("ok") is True; ms=d.get("monitors"); assert isinstance(ms,list) and len(ms)>=1; assert all(m.get("backend") in {"backlight","ddc","sdr"} for m in ms)'
```

## Manual Debug Mode

Use foreground daemon mode only for debugging:

```bash
./bin/omabright daemon
```

This blocks the terminal. Run client commands from another shell:

```bash
./bin/omabright status
./bin/omabright up
./bin/omabright down
```

## Hyprland Keybinds

Omarchy binds brightness keys to `omarchy-brightness-display` by default.
To use `omabright`, add this to `~/.config/hypr/bindings.conf`:

```conf
# Unbind default Omarchy brightness handler
unbind = , XF86MonBrightnessUp
unbind = , XF86MonBrightnessDown
unbind = ALT, XF86MonBrightnessUp
unbind = ALT, XF86MonBrightnessDown

# Bind to omabright (cursor-aware brightness)
bindel = , XF86MonBrightnessUp, exec, omabright up
bindel = , XF86MonBrightnessDown, exec, omabright down
bindel = ALT, XF86MonBrightnessUp, exec, omabright up --step 1
bindel = ALT, XF86MonBrightnessDown, exec, omabright down --step 1
```

## External Monitor Behavior

Not all monitors, docks, and adapters support DDC/CI reliably.

- If DDC works, `omabright` uses hardware brightness (`ddcutil`)
- If DDC is unavailable, `omabright` falls back to Hyprland SDR brightness
- If `ddcutil detect --brief` reports an output as `Invalid display` (common on some docks), `omabright` treats that path as non-DDC and directly uses SDR fallback to avoid repeated timeouts

SDR fallback configuration in `~/.config/omabright/config.json`:
- `sdr_min` / `sdr_max` (default `0.2` to `1.0`)
- `sdr_fallback_enabled` (default `true`)

Reset SDR brightness for external monitors:

```bash
omabright reset
```

## DDC Mapping

When `ddcutil` is available, connector-to-bus mapping is maintained automatically.

- Automatic mapping: DRM connector names from `ddcutil detect --brief`
- Manual override: `ddc_overrides` in `~/.config/omabright/config.json`

## Troubleshooting

Brightness keys do nothing:

```bash
hyprctl binds | rg -n "XF86MonBrightness|omabright"
systemctl --user status omabright.service
omabright ping
```

Dock-connected monitor does not change hardware brightness:

```bash
ddcutil detect --brief
```

If the output is `Invalid display`, that connector path does not support reliable DDC.
Use SDR fallback (default) or change cable/dock path.

DDC permission problems:

```bash
ls -l /dev/i2c-*
id
groups
```

After changing permissions or group membership, re-login and test again.

## Project Structure

- `bin/omabright`: single-file daemon + client CLI
- `systemd/omabright.service`: user service template
