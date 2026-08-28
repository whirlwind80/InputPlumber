# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Fork-only file — do not upstream.** This file and `handoff.md` live only on the `claude` branch
> of this fork (whirlwind80/InputPlumber) and must never be merged, cherry-picked, or otherwise
> included in a PR/branch destined for `upstream` (ShadowBlip/InputPlumber). They contain
> machine-specific dev notes, not project documentation. Feature/PR branches (e.g.
> `fix/zotac-zone-*`) must branch from `main`, not from `claude`, and `claude` must never be merged
> into them.

## What this is

InputPlumber is a Rust daemon for Linux that combines multiple physical input devices (gamepads,
keyboards, mice, IMUs, hidraw devices) into virtual "target" devices, using DBus for configuration
and control. It's commonly used on handheld gaming PCs (Steam Deck, ROG Ally, Zotac Zone, etc.) as
part of distros like Bazzite, to route non-standard hardware buttons into a Steam-compatible input
scheme.

## Build and test commands

- `make build` / `make release` — release build (`cargo build --release`)
- `make debug` — debug build
- `make run` — build debug and run as root with `LOG_LEVEL` / `ENABLE_METRICS` env vars
- `make test` — runs `cargo clippy --all -- -D warnings`, `cargo test -- --show-output`, plus the
  autostart-rules check below. Run this before submitting any change.
- `cargo test <module::path>::<test_name> -- --exact --show-output` — run a single test
- `make test-autostart-rules` — checks `rootfs/usr/lib/udev/rules.d/90-inputplumber-autostart.rules`
  stays in sync with device configs (`cargo test config::config_test::check_autostart_rules`)
- `make test-polkit-usage` — checks polkit actions match policy file
- `make format` — `rustfmt --edition 2021` on all sources; CONTRIBUTING.md requires this before every commit
- `make generate` — regenerates JSON schema files under `rootfs/usr/share/inputplumber/schema/` from
  the config structs; run this after changing any `#[derive(JsonSchema)]` config type
- `make in-docker TARGET='<target>'` — runs any make target inside the project's Docker builder
  image (this is what CI uses for `test` and `dist`)
- Remote/root debugging: InputPlumber needs elevated device access, so local debugging normally
  happens via `lldb-server` on the target device — see CONTRIBUTING.md for the systemd unit and
  VSCode CodeLLDB config. On a device, running with `RUST_LOG=debug` (`LOG_LEVEL=debug` for
  `make run`) foreground is the most direct way to observe device match/grab behavior live.

## Commit conventions (CONTRIBUTING.md)

- Angular/semantic-release style: `fix(scope): message`, `feat(scope): message`, `chore(scope): ...`,
  `docs(scope): ...` — this repo's releases are automated from these.
- No IDE-specific files in the repo.
- Always run `cargo fmt` before committing.
- AI/tool-generated contributions are allowed but must be disclosed in the commit body (e.g.
  `Co-developed-by: Claude Opus 4.6`), with an explanation of what was generated — and you must be
  able to explain every generated line. Using AI to respond to human reviewers is prohibited.

## Architecture

### Pipeline: source devices → CompositeDevice → target devices

1. **`input::manager::Manager`** (`src/input/manager.rs`) is the top-level orchestrator, run from
   `main.rs`. It watches `/dev/input`, udev, DMI data, and CPU info; loads `CompositeDeviceConfig`
   YAML files (from `get_devices_paths()`, i.e. `/usr/share/inputplumber/devices/` overlaid by
   `/etc/inputplumber/devices/`); and for every physical device that appears, checks each config's
   `matches` (DMI/udev conditions) and `source_devices` entries to decide which `CompositeDevice`
   should claim it.
2. **`config::CompositeDeviceConfig`** (`src/config/mod.rs`) is the deserialized form of a device
   YAML (schema: `rootfs/usr/share/inputplumber/schema/composite_device_v1.json`, instances in
   `rootfs/usr/share/inputplumber/devices/*.yaml`). Matching logic for individual source device
   kinds (evdev, hidraw, udev/led, tty, iio) lives here as `has_matching_*` methods, using
   `glob_match` against fields (name, phys_path, handler/sysname, vendor/product id, etc.) reported
   by the kernel/udev — so config field values must exactly match (or glob-match) what the kernel
   actually reports (see `UdevDevice` in `src/udev/device.rs`), not just what's expected.
3. **`input::composite_device::CompositeDevice`** (`src/input/composite_device/mod.rs`, the largest
   module) owns one or more source devices (`input::source::*`, one submodule per source kind:
   evdev, hidraw, iio, led, tty) and one or more target devices (`input::target::*`, one submodule
   per emulated device type: xbox-elite, keyboard, mouse, dualsense, steam_deck, etc.). It reads
   native events from sources, translates them via capability maps/profiles, and forwards to
   targets. `CompositeDeviceConfig.source_devices[].capability_map_id` links a source device group
   to a `CapabilityMapConfig`.
4. **`config::capability_map`** (`src/config/capability_map/`, mappings in
   `rootfs/usr/share/inputplumber/capability_maps/*.yaml`) defines how raw source events (e.g. a
   specific evdev keycode) map onto InputPlumber's internal `Capability` types
   (`src/input/capability.rs`, e.g. gamepad buttons like `QuickAccess`). `capability_map_id` can be
   set at two different scopes, and they behave differently:
   - **Per source** (`source_devices[].capability_map_id`, the common case): `EventDevice::new`
     (`src/input/source/evdev.rs`) loads the map and builds a dedicated `EventTranslator` for that
     one source device only (`src/input/source/evdev/gamepad.rs` etc.) — it only ever sees events
     that source itself received. There's no cross-source leakage here, but if a single
     `source_devices` entry's `evdev.name` glob matches more than one physically distinct device
     (e.g. two devices with similar/overlapping kernel-reported names folded into one entry), that
     one entry's translator applies to *both* identities' events indiscriminately — a mapping
     written for one of them can mistranslate the other's genuine events. This is also what happens
     if the same map (by id) is independently attached to two different `source_devices` entries
     that happen to both emit the same capability — each gets its own translator instance built
     from the same rules, so both mistranslate their own matching events.
   - **Device-wide** (a top-level `capability_map_id` on `CompositeDeviceConfig` itself, not on any
     `source_devices` entry): populates `CompositeDevice`'s own `translatable_capabilities`/
     `capability_map` (`composite_device::mod.rs`'s `load_capability_map`), applied in
     `process_event()` by `Capability` value across *all* sources on that composite device,
     regardless of which source emitted the event. This path only exists (and only matters) when a
     device config actually sets that top-level key — most device configs, including
     `50-zotac-zone.yaml`, don't, so `process_event()`'s device-wide check never fires for them and
     only the per-source behavior above applies. Don't assume the device-wide path is in play
     without checking for that top-level key.
   Also note: for `mapping_type: evdev: chord` entries, an event whose signature is part of *any*
   chord mapping is routed straight into the translator and never appears in a source's per-event
   trace log (e.g. `KeyboardEventDevice::translate()`'s `"Received event"` trace) — a quiet
   `journalctl`, even at `trace`, does not mean the event never arrived.
5. **`drivers/`** contains per-device protocol implementations (parsing/writing hidraw reports,
   IMU access, etc.) for specific hardware, each named after the device/vendor it supports (e.g.
   `zotac_zone`, `rog_ally`, `steam_deck`, `dualsense`). These are used by the corresponding source
   or target modules rather than by `CompositeDevice` directly.
6. **`dbus/interface/`** exposes the manager, composite devices, and individual source/target
   devices over DBus (`org.shadowblip.InputPlumber`), mirroring the `input/` module structure
   (`source/*`, `target/*`). `bindings/dbus-xml/` and `docs/dbus-interface/*.md` are generated from
   the live DBus interface via `make dbus-xml` and `make docs` — don't hand-edit them.

### Config loading and overlay

Device/profile/capability-map configs are loaded from multiple directories in priority order via
`config::path::get_devices_paths()` / `get_multidir_sorted_files()` — typically
`/usr/share/inputplumber/...` (packaged) overlaid by `/etc/inputplumber/...` (local admin
override). When debugging a specific device's matching behavior, check both locations, since a
stale or missing `/etc` override silently falls back to the packaged default.

### Adding support for a new device

Typically requires: a `CompositeDeviceConfig` YAML in `rootfs/usr/share/inputplumber/devices/`
(matching real `dmi_data`/`udev` values — verify with `cat /proc/bus/input/devices`, `udevadm info`,
not just spec sheets), a capability map YAML if the device sends non-standard keycodes, and
possibly a new `drivers/<device>` module + `target`/`source` wiring if the device needs custom
protocol handling (e.g. hidraw report parsing) rather than plain evdev passthrough.

## Debugging input pipeline issues on real hardware

- **Capture at the hidraw layer before evdev.** A raw `hidraw` read needs no root, doesn't require
  stopping InputPlumber, and is unaffected by exclusive `evdev` grabs (hidraw broadcasts to every
  open fd). It's almost always the fastest way to confirm what a device is actually sending.
- **Parse the HID report descriptor instead of guessing bit meanings.** `HIDIOCGRDESC` (mutate the
  buffer if it exceeds 1024 bytes) gives you the bit↔usage mapping directly; kernel HID debugfs at
  `/sys/kernel/debug/hid/<bus:vid:pid.NNNN>/events` (find the id via `ls /sys/bus/hid/devices/` and
  match which one owns the target `eventN` under its `input/inputM/eventN`) shows parsed usage
  names per report, live, independent of any evdev grab — useful ground truth before suspecting
  InputPlumber's own logic.
- **Change one variable at a time when capturing.** Mixed inputs (e.g. a dial spinning while a
  button is also held) in the same capture window produces bit patterns that look like the wrong
  control. Separate suspected controls into distinct timed windows (e.g. 10s of A, 5s idle, 10s of
  B) so timestamps disambiguate them.
- **Search for an existing vendor driver before reverse-engineering a protocol from scratch** —
  handheld vendors' kernel/config protocols are frequently already documented by community driver
  projects (e.g. OpenZotacZone/ZotacZone-Drivers for Zotac). Cross-check any independently-derived
  bit mapping against it.
- **Verify a binary protocol by reading before writing.** Confirm a checksum/framing implementation
  against a captured real frame, then use read-only GET-style commands to confirm request/response
  framing, before ever sending a SET/write command to the device.
- **To get real evdev-level ground truth, you must actually release InputPlumber's grab**: `sudo
  systemctl mask inputplumber && sudo systemctl stop inputplumber` (mask first — a plain `stop` gets
  revived by the udev autostart rule), then `evtest /dev/input/eventN` (or `sudo timeout Ns evtest
  ...` to auto-exit), then `sudo systemctl unmask inputplumber && sudo systemctl start inputplumber`
  to restore. This briefly kills the device's controller/keyboard/mouse function. Bonus: with the
  grab released, you can also see what the desktop/compositor session itself does with a raw key
  combo it would otherwise never see — useful for telling apart "InputPlumber is doing this" from
  "the raw input leaked through and the session's own shortcut fired."
- **Running a locally-built dev binary in place of the packaged service**: `systemctl stop` alone
  isn't enough (the udev autostart rule revives the packaged binary) — `systemctl mask` it first. A
  systemd `ExecStart` drop-in pointing at a home-directory binary gets blocked by SELinux
  (`203/EXEC`); run it manually with `sudo <path-to-binary>` instead. A backgrounded `&` process
  doesn't die on Ctrl+C — find it with `pgrep -fa <path>` and `pkill -9 -f <path>`. Foreground dev
  binary output goes to the terminal, not `journalctl` — redirect explicitly (`> file 2>&1`). Always
  `systemctl unmask` + `systemctl start` when done. `/etc/.../*.d/` overrides must be manually
  re-copied after any repo change — the running daemon reads the deployed files, not the checkout.
  A manually-backgrounded process like this is tied to your login session/cgroup — switching to a
  different session (e.g. Steam Gaming Mode on a handheld) can kill it outright, leaving no input
  daemon running at all. If you need the dev binary to survive a session switch (e.g. to test in
  Gaming Mode), don't rely on a background shell job or an `ExecStart` drop-in (SELinux blocks that
  for a home-directory path); on an ostree/bootc system instead: `sudo rpm-ostree usroverlay` (makes
  `/usr` writable for the current boot only — fully reverts on next reboot), back up the packaged
  binary (`sudo cp /usr/bin/inputplumber /usr/bin/inputplumber.orig`), copy the dev build over it
  (`sudo cp target/debug/inputplumber /usr/bin/inputplumber` — `systemctl stop` first, since
  overwriting a running executable's file fails with "Text file busy"), then a normal `systemctl
  restart inputplumber` runs your code as a real, session-independent systemd service. Restore
  (`systemctl stop`, copy the `.orig` back, `systemctl start`, remove the backup) before finishing,
  even though a reboot alone would also revert it.
- **A dev binary run from inside the source checkout (`cd ~/InputPlumber && ... ./target/debug/...`)
  also checks `./rootfs/usr/share/inputplumber/...` as a relative-path config source**, in addition
  to `/usr/share` and `/etc`. If that checkout's branch predates fixes already deployed to `/etc`
  (e.g. an older branch being tested for something unrelated), its own bundled configs can load as
  an extra, conflicting layer and reintroduce already-fixed bugs (e.g. duplicate `CompositeDevice`s)
  that have nothing to do with what you're actually testing. Installing the dev binary to
  `/usr/bin/inputplumber` (see above) avoids this, since then it only sees `/usr/share` + `/etc`
  like the packaged binary does.
- Other useful spot-checks: `fuser -v /dev/hidrawN` or `/dev/input/eventN` to see who holds a node
  open; `busctl --system get-property org.shadowblip.InputPlumber
  /org/shadowblip/InputPlumber/CompositeDevice0 org.shadowblip.Input.CompositeDevice
  SourceDevicePaths` to see what a composite device currently has attached.
- `pkill -f <pattern>` can kill your own shell if the pattern also matches your current command
  line — prefer a more specific pattern or `pkill -9 -f <full-path>`.

## This machine: keeping the Zotac Zone working across updates

This checkout is used to fix input handling on the Zotac Gaming Zone (G0A1W) it runs on. Two PRs
are open upstream and, until both land in an installed Bazzite image, the device depends on local
workarounds that need occasional attention. Full background is in `handoff.md` (tracked on this
fork's `claude` branch only — see the fork-only note at the top; it is *not* untracked) — that file
is a running session-to-session handoff log (current PR/review status, in-progress experiments,
next-session TODOs); durable facts about the codebase or this device belong here instead.

### Development environment on this machine

This checkout runs inside a toolbox (`inputplumber-dev`). Consequences:
- Run host commands via `flatpak-spawn --host <command>` — this includes anything checking
  `/etc/inputplumber/...`, since the toolbox's own `/etc` is the *container's*, not the host's.
- `sudo` does not work non-interactively inside the toolbox session — hand any command needing
  `sudo` to the user to run themselves (e.g. via Claude Code's `!` prefix) rather than attempting it
  directly.

- **PR #664** — Steam/QAM buttons, View button, duplicate composite device, the dial capability
  mappings that were swallowing the left touchpad's scroll events, and the physical HOME button's
  real key chords (mapped to `Screenshot`/`Guide` — see Hardware reference below).
- **PR #668** — binds the rear paddles by speaking the vendor config protocol over hidraw. Their
  mapping ships empty, so the firmware sends nothing for them until it is written.

### What each fix currently depends on

| Working because of | Survives a Bazzite update? |
|---|---|
| `/etc/inputplumber/devices.d/50-zotac-zone.yaml` | Yes — admin-added files under `/etc` are kept by rpm-ostree's 3-way merge |
| `/etc/inputplumber/capability_maps.d/zone_type1.yaml` | Yes, same |
| Paddle mapping written into the device | No — it is volatile by design and lost whenever the device loses power |

Both override files must stay in sync with this checkout's `rootfs/usr/share/inputplumber/...`
copies; re-copy them after changing either, since InputPlumber reads the deployed files rather
than the repo. Note the `.d` suffix: overrides are only read from `devices.d/` and
`capability_maps.d/`, never from `devices/` or `capability_maps/`.

### Hardware reference (0x1ee9:0x1590, all `hid-generic`)

| hidraw node | USB interface | Contents |
|---|---|---|
| `/dev/hidraw0` | 1 | rid=2 keyboard (F16-F20 + the real HOME chords), rid=3 dial 24-bit array, rid=4 mouse/touchpad, rid=5/6 |
| `/dev/hidraw1` | 2 | rid=7/8/9 (8-byte status). Not observed to actually transmit |
| `/dev/hidraw2` | 3 | **Command interface** — 64-byte in/out, no report id. Sends `e1 00 XX 3c b2 …` telemetry every 2s. See vendor config protocol below |

evdev nodes (interface 1): `event2`/`event5`/`event6` = bare "...ZOTAC GAMING ZONE" (limited,
duplicate-composite-device trap fixed by adding `phys_path`); `event3` = "...ZONE Keyboard" (F16-F24,
`KEY_HOME`/`KEY_END` paddles once remapped, the real HOME chords); `event4` = "...ZONE Mouse"
(`REL_X`/`REL_Y`/`REL_WHEEL`/`REL_WHEEL_HI_RES` — **no `REL_HWHEEL`**; confirmed 2026-08-28 via
`evtest`'s declared-capabilities list and a live capture of a deliberately horizontal stroke on the
left pad: only `REL_WHEEL` ever fires, so this hardware has no genuine horizontal-scroll axis at
all — a prior version of this doc claimed `REL_HWHEEL` existed here, which was wrong); `event14`/
`js0` = "ZOTAC Gaming Zone" (kernel `xpad`, interface 1.0) — the real gamepad functionality
(sticks/ABXY/shoulders/triggers/D-pad) lives here and must be a `source_devices` entry or Steam
reads it raw, ungrabbed. Node numbers are not stable across reboots/replugs (observed as `event10`
on 2026-08-28) — match by `phys_path`/name, never assume a fixed number.

**Separate InputPlumber core bug (not Zotac-specific), found while investigating the above**: in
`src/input/event/evdev.rs`, `EvdevEvent::as_capability()` maps *both* `REL_WHEEL` and `REL_HWHEEL`
to the same `Capability::Mouse(Mouse::Wheel)`, and `get_value()` returns a bare `InputValue::Float`
for either (not a `Vector2`), so the source axis is lost. On output, `event_codes_from_capability`
for `Mouse::Wheel` lists *both* `REL_WHEEL` and `REL_HWHEEL`, and the `Float` value gets written to
whichever code is being emitted — so every `Mouse::Wheel` event a source produces is mirrored onto
*both* output axes with the same magnitude, regardless of which axis it actually came from. On any
device with real independent `REL_WHEEL`/`REL_HWHEEL` this would make every vertical scroll tick
inject a phantom horizontal one and vice versa. Worth a separate upstream issue; out of scope for
the Zotac Zone PRs.

⚠️ **The code bug above is real (read directly from the source), but do not use it to explain the
"scrolling goes sideways/diagonally" feel on this device — that link was tested on 2026-08-28 and
ruled out.** The same behaviour persists with the `group: mouse` entry removed entirely, i.e. with
InputPlumber nowhere in the path. The actual cause is the pad itself: it only ever emits `REL_WHEEL`,
and swiping *across* the strip's own axis makes it emit vertical ticks in an erratic mixed up/down
direction (clearly visible as alternating `+1`/`-1` in the raw captures) rather than nothing. That
erratic vertical scrolling is what reads as "wrong direction". Cite the code bug from the code only.

Button → signal facts:
- Steam button → `KEY_F17` (hidraw0 rid=2: `02 09 00 6c`) → `Guide`
- QAM button → `KEY_F18` (`02 09 00 6d`) → `QuickAccess`
- View button → `BTN_SELECT` (event14)
- Left paddle (M2) / right paddle (M1) → after `CMD_SET_BUTTON_MAPPING` remap, `KEY_HOME`/`KEY_END`
  (`02 00 00 4a` / `02 00 00 4d`) → `LeftPaddle1`/`RightPaddle1`
- Dials → on this `hid-generic`-only machine (no vendor kernel module loaded — `modinfo
  zotac_zone_hid` finds nothing, all three USB interfaces bind `hid-generic`), the dials never
  surface as evdev at all; they only reach hidraw0 rid=3. Byte `[3]` bits: `0x01` right CW / `0x02`
  right CCW / `0x08` left CW / `0x10` left CCW (matches vendor driver `zotac-zone-hid-core.c`'s
  `ZOTAC_{RIGHT,LEFT}_DIAL_{CW,CCW}_BIT`). Each detent arrives as a press+release pulse. Reading
  them needs a dedicated hidraw source parsing rid=3 directly (or the vendor module, which does
  expose them as evdev on its own separate `wheel_input` device — see below).
  **Do not** attach a dial-targeting capability_map (`REL_HWHEEL`/`REL_WHEEL` → dial capability) to
  the *touchpad's* `source_devices` entry, or to any entry whose `evdev.name` glob also matches the
  touchpad. The left touchpad (a scroll-only surface, separate from the right pointer touchpad)
  emits genuine `REL_WHEEL` of its own (only `REL_WHEEL` — no `REL_HWHEEL` on this hardware; see the
  corrected node note above and the related InputPlumber core Wheel-duplication bug it explains).
  `capability_map_id` on a `source_devices` entry
  builds a translator scoped to *that entry's own* events (see the capability_map note in
  Architecture above) — so this isn't a device-wide leak, but if that entry is the touchpad (or a
  name glob that resolves to the touchpad on hardware without the vendor module), its own
  translator will still mistranslate its own genuine wheel events into the dial capability. This is
  exactly what happened once: the touchpad source simply had the same `capability_map_id` as the
  dial rules. The dial mapping now lives in its own map (`zone_type1_dial.yaml`, id `zone1_dial`),
  intentionally left unattached to any `source_devices` entry until something (vendor module or a
  hidraw parser) actually produces dial-only evdev events to attach it to — the touchpad's own
  entry must never reference it, directly or via a shared/overlapping name glob. Cross-checked
  against `zotac-zone-hid-core.c` from OpenZotacZone/ZotacZone-Drivers: with the vendor module
  loaded, the dials do get their own separate `wheel_input` evdev device (distinct from the
  touchpad's `mouse_input`) — but `mouse_input`'s own report handler *also* independently emits
  `REL_WHEEL` for the touchpad's native scroll (unrelated to the dials), so the same
  name-glob-conflation risk applies even with the vendor module present if a `source_devices` entry
  ends up matching both devices under one name pattern.
- Physical HOME button sends chords outside the F16-F19 scheme entirely: short press =
  `KEY_LEFTMETA`+`KEY_D`, long press = `KEY_LEFTCTRL`+`KEY_LEFTALT`+`KEY_KPDOT`. Mapped via
  `mapping_type: evdev: chord` to `Screenshot`/`Guide` respectively — these are the only two buttons
  in this device's capability_map with real, special-cased evdev output on the `xbox-elite` target
  (`event_codes_from_capability` in `src/input/event/evdev.rs`, `write_event` in
  `src/input/target/xpad.rs`); a `unified_gamepad`/DBus target is not attached for this device, so
  anything mapped to `QuickAccess2` or `Keyboard` here is a silent no-op. **Note**: consuming these
  chords means they no longer pass through to the desktop/gamescope session's own keyboard target,
  so KDE's `Meta+D` "Show Desktop" shortcut stops firing in desktop mode too (capability_map
  translation doesn't distinguish session type). This tradeoff was deliberately accepted for this
  device.

### Vendor config protocol (interface 3 = `/dev/hidraw2`)

Source: [OpenZotacZone/ZotacZone-Drivers](https://github.com/OpenZotacZone/ZotacZone-Drivers)
`driver/hid/zotac-zone-hid-config.c` — GPL-2.0-or-later, Copyright (c) 2025 Luke D. Jones
(flukejones). InputPlumber is GPL-3.0-or-later, so borrowing under the "or-later" clause is fine —
keep the source/author attribution.

Frame: 64 bytes (a leading `0x00` report-number byte is prepended on hidraw write, 65 total).
`[0x00]` header tag `0xE1`, `[0x01]` reserved `0x00`, `[0x02]` sequence (increments per command,
outside the CRC range), `[0x03]` payload size `0x3C`, `[0x04]` command, `[0x05..]` data,
`[0x3E..0x40)` CRC (big-endian) over bytes `4..0x3D`:

```python
def crc(buf):
    c = 0
    for i in range(4, 0x3E):
        h1 = (c ^ buf[i]) & 0xFF
        h2 = h1 & 0x0F
        h3 = ((h2 << 4) ^ h1) & 0xFFFFFFFF
        h4 = h3 >> 4
        c = ((((((h3 << 1) ^ h4) << 4) ^ h2) << 3) ^ h4 ^ (c >> 8)) & 0xFFFF
    return c
```

Commands: `CMD_SET_BUTTON_MAPPING 0xA1`, `CMD_GET_BUTTON_MAPPING 0xA2`, `CMD_GET_DEVICE_INFO 0xFA`,
`CMD_SAVE_CONFIG 0xFB` (persists to the device — do not send unless that's actually intended),
`CMD_RESTORE_PROFILE 0xF1`, `CMD_SET_PROFILE 0xB1`. `BUTTON_M1 = 0x01`, `BUTTON_M2 = 0x02`,
`BUTTON_MAX = 0x18`; HID keycodes `home = 0x4a`, `end = 0x4d`. Button ids `0x03`-`0x0A` are
touchpad-edge (L/R_TOUCH_{UP,DOWN,LEFT,RIGHT}), `0x0B`-`0x18` are LB/RB/LT/RT/ABXY/D-pad/LS/RS.
Dials aren't in this table — not remappable this way, only readable via rid=3 directly.

`SET_BUTTON_MAPPING` payload (14 bytes from frame offset `0x05`): `[0]` source button id, `[1..4]`
gamepad button bitfield, `[5]` modifier key, `[6]` reserved, `[7..12]` 6 HID keyboard keys, `[13]`
mouse button bitfield. Response: `[0x04]` echoes the command code; `SET_BUTTON_MAPPING`'s status is
at `[0x06]` (`0` = OK). GET-command response data generally starts at `[0x05]`, except
`GET_DEVICE_INFO`'s, which in practice starts at `[0x06]` — one byte off from what the kernel
driver source documents.

### Re-binding the paddles

Any full power loss clears the paddle mapping, because the installed InputPlumber predates #668 and
so nothing re-applies it at startup. Re-run:

```sh
python3 ~/zotac-zone-tools/zotac-zone-paddles
```

It finds the right hidraw node itself and needs no root. `~/zotac-zone-tools/zotac-zone-paddles.sh`
is a wrapper for registering it as a non-Steam game, so it can be launched from Game Mode.

### After the PRs are merged and shipped — remove the workarounds

The `/etc` overrides take priority over the packaged configs *permanently*. Once upstream's
versions ship, ours keep shadowing them, so any changes made during review, and any later upstream
work on this device, would never take effect. A config schema change could also leave a stale
override failing to load.

Check whether the fixes have landed:

```sh
grep -c "phys_path" /usr/share/inputplumber/devices/50-zotac-zone.yaml   # >= 1 means #664 shipped
```

When they have:

```sh
sudo rm -rf /etc/inputplumber
sudo systemctl restart inputplumber
rm -rf ~/zotac-zone-tools          # scripts are then obsolete too
```

and remove the non-Steam shortcut from Steam.
