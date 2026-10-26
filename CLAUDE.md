# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A custom home screen for a **rooted** LG webOS TV, plus ad/tracking removal and auto-launch watchers. There is no build system, package manager, or test suite. Every file is deployed by hand onto the TV (over telnet/SSH as root) and runs there, not on the dev machine. `README.md` is the authoritative setup guide.

## Layout and how the pieces connect

- `custom_home/index.html`: a single self-contained webOS web app (inline CSS/JS, inline SVG icons). It launches other apps and HDMI inputs through `PalmServiceBridge` calls to `luna://com.webos.applicationManager/launch` (`callLuna` / `launchApp`). App IDs, tiles, and the HDMI mapping are hardcoded arrays near the bottom of the script and are specific to the author's TV.
- `watchers/homekey_watcher.py`: passively reads raw `input_event` structs (`<IIHHi`) from `/dev/input/event1` and launches the app when it sees Home key code `773`. It must **never grab** the device exclusively, because a crashed daemon would otherwise leave the remote dead.
- `watchers/powerstate_watcher.py`: holds a persistent `luna-send -i` subscription to `com.webos.service.tvpower/power/getPowerState` and launches the app when the state changes to `Active`.
- Both watchers close the app (`closeByAppId`) before launching it, because a plain `launch` only resurfaces a stale instance. Both use `TARGET_APP = "com.naveen.customhome"`, which must match the installed app ID.
- `ads/disable_ads.sh` / `ads/undo_ads.sh`: bind-mount an empty dir over 18 `com.webos.app.*` packages. Never write to the signed read-only partition; keep these idempotent (`mountpoint -q` guards) and keep undo symmetric with disable.
- `init.d/README.md`: explains persistence through Homebrew Channel's `/var/lib/webosbrew/init.d/NN-name.sh` one-liner scripts.

## Constraints when editing

- Target runtime is an embedded webOS device: BusyBox shell (`/bin/sh`, not bash), an old on-device `python3` (stdlib only, no pip), and an old Chromium-based web engine. Keep JS conservative (the existing code uses `var` / `function` and no modules or build step) and keep the HTML a single file.
- Device-specific values (input device path, key code, app IDs, HDMI names) are intentionally hardcoded. When changing them, keep the README's instructions for finding your own values accurate.
- Scripts can't be run locally to test. At most, check syntax (`python3 -m py_compile watchers/*.py`, `sh -n ads/*.sh`) and open `custom_home/index.html` in a browser for layout (luna calls are no-ops off-device).
- `appinfo.json` isn't in the repo. If you add one, use `"trustLevel": "default"`, not `"trusted"` (see README).
