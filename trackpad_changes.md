# Trackpad Changes

## Hyprland touchpad settings

Changed `/home/mark/.config/hypr/input.conf`:

```conf
touchpad {
  clickfinger_behavior = true
  tap-to-click = false
  scroll_factor = 0.4
  disable_while_typing = true
}
```

Effect:

- Disables tap-to-click, including two-finger tap.
- Keeps physical/clickfinger clicking enabled.
- Enables Hyprland's built-in touchpad disable-while-typing behavior.

Hyprland was reloaded and `hyprctl configerrors` returned clean.

Backup created:

```text
/home/mark/.config/hypr/input.conf.bak.1780029897
```

## Trackpad dead-zone filter

Cloned:

```text
/home/mark/disable-trackpad-area
```

Source:

```text
https://github.com/hamza72x/disable-trackpad-area
```

The upstream filter originally only searched for `Apple SPI Trackpad`. This MacBook exposes the real trackpad as:

```text
Apple Inc. Apple Internal Keyboard / Trackpad
```

So `/home/mark/disable-trackpad-area/trackpad-filter.c` was patched to detect that device name, while requiring multitouch axes so it picks the trackpad event node and not the keyboard event node.

The filter was also extended with:

```text
--left-from PCT
--left-to PCT
```

These restrict the left dead zone to a vertical range instead of always disabling the full left edge.

## Current target disabled area

Current local service file:

```text
/home/mark/disable-trackpad-area/trackpad-filter.service
```

Current target command:

```bash
/usr/local/bin/trackpad-filter --left 18 --right 0 --top 0 --left-from 0 --left-to 85
```

Meaning:

- Disable the leftmost 18% of the trackpad.
- Start at the top edge: `--left-from 0`.
- Continue down to 85% of the trackpad height: `--left-to 85`.
- Do not disable the right edge.
- Do not disable a separate top strip.

ASCII view:

```text
top
┌────────────────────────────┐
│XXXXX│                      │  0%
│XXXXX│                      │
│XXXXX│                      │
│XXXXX│  disabled area       │
│XXXXX│                      │
│XXXXX│                      │
│XXXXX│                      │
│XXXXX│                      │
│XXXXX│                      │
├─────┘                      │  85%
│                            │
└────────────────────────────┘
bottom                         100%
```

## Applying changes

Because installing the filter writes to `/usr/local/bin` and `/etc/systemd/system`, apply changes from an interactive terminal:

```bash
cd ~/disable-trackpad-area
sudo ./install.sh
```

The installer builds the binary, copies it to `/usr/local/bin/trackpad-filter`, installs `/etc/systemd/system/trackpad-filter.service`, enables the service, and restarts it.

Check status:

```bash
systemctl status trackpad-filter.service --no-pager
```

A working service should stay:

```text
Active: active (running)
```

Useful logs:

```bash
journalctl -u trackpad-filter.service -n 100 --no-pager
```

The logs should show the detected trackpad and computed active area. If they show `Error: cannot find Apple SPI Trackpad`, the installed binary is not the patched version.

## Adjusting the area

Edit:

```text
/home/mark/disable-trackpad-area/trackpad-filter.service
```

Then rerun:

```bash
cd ~/disable-trackpad-area
sudo ./install.sh
```

Common adjustments:

```bash
# Wider left strip
--left 22

# Narrower left strip
--left 14

# Disable the full left edge
--left-from 0 --left-to 100

# Disable only upper/middle left edge
--left-from 0 --left-to 70
```
