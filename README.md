# omabright (Omarchy/Hyprland per-monitor brightness)

Goal: wherever your mouse cursor is, your laptop brightness keys adjust that monitor.

## Requirements

- Required: `hyprctl`, `brightnessctl`, `systemd --user`
- External monitors (recommended): `ddcutil` (DDC/CI for real monitor brightness)
  - Commonly also needed: allow your user to access `/dev/i2c-*` (e.g. udev `uaccess` or `i2c` group)
- OSD (enabled by default): `swayosd-client` (ships with Omarchy; shows the bar on the target monitor)

## External Monitor Fallback

Not every monitor/dock/adapter path supports DDC/CI reliably. When DDC is unavailable, `omabright` falls back to Hyprland `sdrBrightness` (software dimming, not the monitor backlight).

- Range is controlled by `sdr_min/sdr_max` in `~/.config/omabright/config.json` (default `0.2..1.0`)
- If the image looks wrong, run `omabright reset` to reset external monitors' `sdrBrightness`

## Quick Start

1) Create default config (optional)

```bash
omabright init
```

2) Start the daemon (foreground)

```bash
omabright daemon
```

3) Test from another terminal

```bash
omabright status
omabright up
omabright down
```

## systemd User Service (Recommended)

1) Install to `~/.local/bin` and write the user service

```bash
omabright install
```

2) Enable and start

```bash
systemctl --user enable --now omabright.service
```

## Hyprland Brightness Keybinds (Example)

Omarchy binds brightness keys to `omarchy-brightness-display` by default. In your `~/.config/hypr/bindings.conf`, `unbind` first and rebind to `omabright`:

```
unbind = , XF86MonBrightnessUp
unbind = , XF86MonBrightnessDown
unbind = ALT, XF86MonBrightnessUp
unbind = ALT, XF86MonBrightnessDown

bindeld = , XF86MonBrightnessUp, Brightness up (cursor monitor), exec, omabright up
bindeld = , XF86MonBrightnessDown, Brightness down (cursor monitor), exec, omabright down
bindeld = ALT, XF86MonBrightnessUp, Brightness up precise (cursor monitor), exec, omabright up --step 1
bindeld = ALT, XF86MonBrightnessDown, Brightness down precise (cursor monitor), exec, omabright down --step 1
```

## DDC Mapping

When `ddcutil` is available, the daemon maps Hyprland outputs (e.g. `HDMI-A-1`) to `ddcutil` I2C buses automatically:

- Prefer mapping via `DRM connector` from `ddcutil detect --brief`
- If needed, pin a mapping in `~/.config/omabright/config.json` using `ddc_overrides`

## Files

- `bin/omabright`: single-file script (daemon + client)
- `systemd/omabright.service`: user service template (`omabright install` installs to `~/.config/systemd/user/`)
