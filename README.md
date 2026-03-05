# omabright

Per-monitor brightness control for Omarchy/Hyprland. Your brightness keys automatically adjust whichever monitor your cursor is on.

## Features

- **Cursor-aware brightness**: Brightness keys adjust the monitor under your cursor
- **Laptop display**: Native backlight control via `brightnessctl`
- **External monitors**: Hardware brightness via DDC/CI (`ddcutil`)
- **Software fallback**: Hyprland `sdrBrightness` when DDC/CI is unavailable
- **Visual feedback**: On-screen display (OSD) shows brightness changes on the target monitor
- **Daemon architecture**: Socket-based IPC for instant response

## Requirements

**Core dependencies:**
- `hyprctl` (Hyprland compositor)
- `brightnessctl` (laptop backlight control)
- `systemd --user` (daemon management)

**External monitor support (recommended):**
- `ddcutil` — DDC/CI protocol for hardware brightness control
- User access to `/dev/i2c-*` devices (via udev `uaccess` tag or `i2c` group membership)

**On-screen display (enabled by default):**
- `swayosd-client` — ships with Omarchy, displays brightness bar on the active monitor

## External Monitor Fallback

Not all monitors, docks, or adapters support DDC/CI reliably. When hardware brightness control is unavailable, `omabright` automatically falls back to Hyprland's `sdrBrightness` (software-based dimming).

**Configuration:**
- Brightness range: `sdr_min` / `sdr_max` in `~/.config/omabright/config.json` (default: `0.2` to `1.0`)
- Reset if needed: `omabright reset` restores external monitors to full brightness (`sdrBrightness = 1.0`)

## Quick Start

**1. Create default configuration (optional)**

```bash
omabright init
```

**2. Start the daemon**

```bash
omabright daemon
```

**3. Test brightness control** (from another terminal)

```bash
omabright status    # Show current monitor and brightness
omabright up        # Increase brightness
omabright down      # Decrease brightness
```

## systemd User Service (Recommended)

**1. Install the service**

```bash
omabright install
```

This installs `omabright` to `~/.local/bin` and creates the systemd user service.

**2. Enable and start**

```bash
systemctl --user enable --now omabright.service
```

The service is attached to `graphical-session.target`, so it starts only after the graphical session is ready and stops on logout.

**Upgrade existing installs**

If you installed an older version, reinstall the service template and reload systemd:

```bash
omabright install
systemctl --user daemon-reload
systemctl --user restart omabright.service
```

## Hyprland Keybinds

Omarchy binds brightness keys to `omarchy-brightness-display` by default. To use `omabright` instead, add this to your `~/.config/hypr/bindings.conf`:

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

The `ALT` variants use `--step 1` for fine-grained control.

## DDC/CI Mapping

When `ddcutil` is available, the daemon automatically maps Hyprland outputs (e.g., `HDMI-A-1`) to DDC/CI I2C buses:

- **Automatic detection**: Matches outputs via DRM connector information from `ddcutil detect --brief`
- **Manual override**: Pin specific mappings in `~/.config/omabright/config.json` using the `ddc_overrides` field

This ensures brightness commands target the correct physical monitor.

## Project Structure

- `bin/omabright` — Single-file script (daemon + client)
- `systemd/omabright.service` — systemd user service template (installed to `~/.config/systemd/user/` via `omabright install`)

## Troubleshooting

**Brightness keys do nothing**

- Verify keybinds: `hyprctl binds | rg -n "XF86MonBrightness|omabright"`
- Verify daemon: `systemctl --user status omabright.service`
- Verify IPC: `omabright ping`

If this still fails right after login, run `omabright install` and restart the user service to pick up the latest unit and startup-race fixes.
