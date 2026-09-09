# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Fork-only file — do not upstream.** This file and `handoff.md` live only on the `claude` branch
> of this fork (whirlwind80/InputPlumber) and must never be merged, cherry-picked, or otherwise
> included in a PR/branch destined for `upstream` (ShadowBlip/InputPlumber). They contain
> machine-specific dev notes, not project documentation. Feature/PR branches (e.g.
> `fix/zotac-zone-*`) must branch from `main`, not from `claude`, and `claude` must never be merged
> into them.
>
> **Never write `#<number>` in a commit message on the `claude` branch** (e.g. `PR #668`,
> `issue #655`). GitHub auto-links any `#N`/`ShadowBlip#N` in a commit message pushed to this public
> fork to that PR/issue's timeline as "referenced this pull request" — even though the commit
> itself never touches that PR's branch, it makes this candid, unreviewed session log one click away
> from anyone reading the actual PR (including the upstream reviewer). Write `PR 668` or
> `issue 655` instead (no `#`).

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

⚠️ **"Overlay" here does NOT mean the `/etc` file shadows/replaces the same-named packaged file.**
`get_multidir_sorted_files()` (`src/config/path.rs`) *concatenates* the directory listings and only
uses directory priority as a tiebreaker when sorting by filename — it never drops an entry.
`load_device_configs()` (`src/input/manager.rs`) then parses every path with no dedup by filename
or by the parsed `name:` field, so `/etc/inputplumber/devices.d/50-foo.yaml` and
`/usr/share/inputplumber/devices/50-foo.yaml` end up as **two separate `CompositeDeviceConfig`
entries with the same `name:`**. Per-device matching returns on the first config that matches, and
`/etc` sorts first, so this is usually invisible — until some real device matches a
`source_devices` entry that exists only in the *packaged* copy (e.g. an entry the override has
commented out). That device then finds no home in the `/etc`-based composite device, falls through,
matches the packaged config instead, and **spawns a whole second CompositeDevice** (with its own
duplicate set of target devices — two virtual controllers in Steam, and two composite devices
fighting over the same physical sources). An `/etc` override must therefore be a **matching
superset** of the packaged config's `source_devices` coverage, not a subset. Since which source
lands in which instance depends on udev arrival order, the split looks nondeterministic across
restarts even though the manager's own processing is fully serialized.

Related per-source-entry knobs in `config::SourceDevice` (`src/config/mod.rs`), all easy to miss:

- **`unique`** (defaults to **`true`**): if a second physical device matches a `source_devices`
  entry that an existing composite device already consumed, the default *rejects* it and creates
  another composite device instead of merging. Set `unique: false` on any entry that can legitimately
  match more than one node (several shipped configs — `50-legion_go.yaml`, `50-rog_ally.yaml`,
  `60-flydigi_vader_4_pro.yaml` — already do this).
- **`events: {include: [...], exclude: [...]}`**: per-source event filtering, applied in
  `SourceDriver` (`src/input/source/mod.rs`). It filters on the **translated `Capability` string**
  (`Capability::from_str`, e.g. `"Gamepad:Button:RightPaddle1"`, `"Keyboard:KeyF16"`), *not* raw
  evdev code names, and unparseable strings are silently dropped by a `filter_map`. `exclude: ["*"]`
  plus an `include:` list is the shipped idiom for "only these events" (see `50-ayaneo_*.yaml`).
- `blocked`, `ignore`, `passthrough` also exist and are documented inline in that struct.

Note also that `filtered_events:` appears at the bottom of every shipped capability-map YAML but is
**not a field on `CapabilityMapConfigV2`** — serde ignores it. It does nothing; use the per-source
`events.exclude` above instead.

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
were submitted upstream to fix it in InputPlumber itself, but **both were closed unmerged on
2026-09-02/09-03** — the reviewer (`pastaq`) decided after real-hardware testing that a
config/hidraw-level fix can't fully cover this device (the gamepad reportedly fails to enumerate at
all, without a udev rule for `xpad`, on hardware without the vendor kernel driver present) and that
the real fix is to ship the vendor kernel driver (`zotac_zone_hid`,
[OpenZotacZone/ZotacZone-Drivers](https://github.com/OpenZotacZone/ZotacZone-Drivers)) in OGC
(Universal Blue's kernel) instead.

**★ That happened: OGC shipped `zotac_zone_hid` in the Bazzite 44.20260907 update (kernel
`7.2.3-ogc3.1.fc44`, InputPlumber bumped 0.78.0-5 → 0.79.0-4), and it was first seen on this machine
on 2026-09-08.** The re-verification session that the rest of this document anticipated has been
done — see "After the vendor driver landed" below for what actually changed, which is a lot: the
hardware's evdev topology, the F16-F19 button codes, and which node the real gamepad data comes from
are all different now. Sections written before that date describe the old `hid-generic` world and
are kept only for history; where they conflict with "After the vendor driver landed", that section
wins. Full background on the (still closed) PRs — the review back-and-forth, the closing comments,
exact reasoning — is in `handoff.md`'s 2026-09-04 session
(tracked on this fork's `claude` branch only — see the fork-only note at the top; it is *not*
untracked) — that file is a running session-to-session handoff log (current PR/review status,
in-progress experiments, next-session TODOs); durable facts about the codebase or this device belong
here instead.

### Development environment on this machine

This checkout runs inside a toolbox (`inputplumber-dev`). Consequences:
- Run host commands via `flatpak-spawn --host <command>` — this includes anything checking
  `/etc/inputplumber/...`, since the toolbox's own `/etc` is the *container's*, not the host's.
- `sudo` does not work non-interactively inside the toolbox session — hand any command needing
  `sudo` to the user to run themselves (e.g. via Claude Code's `!` prefix) rather than attempting it
  directly. In practice `flatpak-spawn --host sudo <cmd>` *does* succeed while the host's sudo
  timestamp is still warm (i.e. shortly after the user has authenticated once), which makes it look
  reliable and then fail later — don't build a long unattended sequence on it.
- Every `sudo` run through `flatpak-spawn --host` prints
  `ksshaskpass: Unable to parse phrase "[sudo] password for ..."` first. **That line is noise, not a
  failure** — the command still runs. But wrapping the payload as `sudo bash -c '...'` *does* break
  (the askpass helper takes over and the quoted script gets mangled); keep each `sudo` invocation a
  single plain command instead.
- Claude Code's `!` prefix runs inside the toolbox, so a bare `systemctl ...` there fails with
  "Failed to connect to system scope bus"; it needs the same `flatpak-spawn --host` prefix.
- Driving an interactive TUI (`inputplumber device N test`) or a long `evtest` capture through this
  session's tooling does not work well — backgrounded captures and the user's button presses never
  lined up. Hand the user a plain foreground command in their own terminal and ask them to report
  what they saw; that was the only reliable loop.
- ⚠️ **The checked-out `claude` branch is not the version that is installed.** `claude` sits on
  0.78.1 while the packaged daemon this machine actually runs is newer (0.79.0-4 as of 2026-09-09),
  and local `main` tracks `upstream/main` (0.79.2). Reading `src/...` from the working tree to
  explain live behaviour has already produced a wrong-version answer once. Read the installed
  version's code instead: `git show main:src/input/target/xpad.rs` or `git show v0.79.0:<path>`.
  Local `main` is kept fast-forwarded to `upstream/main` for exactly this; **never merge `main`
  into `claude`** (that is what would leak these fork-only docs toward upstream).

- **PR #664** (closed unmerged) — Steam/QAM buttons, View button, duplicate composite device, the
  dial capability mappings that were swallowing the left touchpad's scroll events, and the physical
  HOME button's real key chords (mapped to `Screenshot`/`Guide` — see Hardware reference below).
- **PR #668** (closed unmerged) — binds the rear paddles by speaking the vendor config protocol over
  hidraw. Their mapping ships empty, so the firmware sends nothing for them until it is written.

Both branches/commits remain useful as local reference (the config fixes, the dial protocol, the
paddle hidraw implementation) even though neither is headed upstream anymore — see handoff.md's
2026-09-04 session for the closing reasoning in full.

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

### Hardware reference (0x1ee9:0x1590) — pre-vendor-driver (`hid-generic`), HISTORICAL

⚠️ **Superseded as of 2026-09-08.** All three HID interfaces now bind `zotac_zone_hid`, which
changes the evdev node layout *and* the button→keycode table below. Read "After the vendor driver
landed" further down for the current values; this subsection is kept because the hidraw-level facts
(report IDs, the config protocol, the dial bit layout) are still accurate and because the machine
could in principle boot a kernel without the driver again.

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

Button → signal facts (⚠️ **the F-key assignments below shifted by one under the vendor driver** —
see "After the vendor driver landed"; the hidraw scancodes are unchanged):
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
  `src/input/target/xpad.rs`); anything mapped to `QuickAccess2` or `Keyboard` here produces nothing
  in practice — but see "Why `QuickAccess2`/`Keyboard` do nothing here" below for the actual reason,
  which is *not* "no DBus target is attached". **Note**: consuming these
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

### Re-binding the paddles — HISTORICAL, no longer needed

⚠️ **Obsolete since 2026-09-08** — the vendor driver applies the paddle mapping itself now; see
"After the vendor driver landed" below. Under `hid-generic`, any full power loss cleared the
paddle mapping and `~/zotac-zone-tools/zotac-zone-paddles` (plus its `.sh` wrapper, registered as
a non-Steam game so it could be launched from Game Mode) re-applied it over the vendor config
protocol. **Both scripts were retired to `~/zotac-zone-tools/obsolete/` on 2026-09-09** and cannot
work any more regardless: InputPlumber holds `/dev/hidraw2` open and the node is now root-only.
The protocol implementation inside them is still a useful reference, as are `crc.py`, `setmap.py`
and `clearmap.py` alongside them.

### After the vendor driver landed (2026-09-08) — current state of this machine

`zotac_zone_hid` arrived with Bazzite `44.20260907` (kernel `7.2.3-ogc3.1.fc44`), bound to all three
HID interfaces (`0003:1EE9:1590.0001/.0002/.0003`, confirmed via `lsmod | grep zotac` and
`readlink /sys/bus/hid/devices/*1EE9:1590*/driver`). InputPlumber went to `0.79.0-4` in the same
update and now **ships its own native `50-zotac-zone.yaml` and `zone_type1.yaml`** built for the
vendor-driver topology — which does *not* make the `/etc` override redundant (see below).

**Current evdev/hidraw topology** (all four vendor nodes hang off HID interface `.0001`, so they all
report `phys_path` `*/input1` — `phys_path` no longer distinguishes them, only the name does):

| Node (numbers move — match by name) | Name | Notes |
|---|---|---|
| `event3` | `ZOTAC Gaming Zone Keyboard` | F16-F19 + `KEY_HOME`/`KEY_END` paddles. **All the special buttons come from here.** |
| `event7` | `ZOTAC Gaming Zone Dials` | The real dials at last: `REL_HWHEEL` (left) / `REL_WHEEL` (right), own device, separate from the touchpad |
| `event9` | `ZOTAC Gaming Zone Mouse` | Touchpads (`REL_X`/`REL_Y`/`REL_WHEEL`). Left ungrabbed on purpose |
| `event10` / `js0` | `ZOTAC Gaming Zone Gamepad` | ⚠️ **Declares a full gamepad capability set but never emits any of it** |
| `event6` / `js1` | `ZOTAC Gaming Zone` | Kernel `xpad` on USB **interface 0** (`phys_path` `*/input0`). **This is still where the real ABXY/sticks/triggers/D-pad/shoulders come from.** |
| `/dev/hidraw2` | — | Config interface, now `crw-------` (InputPlumber hides it) |

⚠️ **The single most expensive trap of the re-verification session**: the vendor driver's own
`ZOTAC Gaming Zone Gamepad` node looks exactly like the device you want — right name, full
`BTN_SOUTH`…`BTN_THUMBR` + `ABS_X/Y/Z/RX/RY/RZ` + `ABS_HAT0X/Y` + `BTN_TRIGGER_HAPPY1-6` capability
list, and it even accepts force-feedback effect uploads — but pressing ABXY on it produces **nothing
at all** (verified with `evtest` on 2026-09-08 with InputPlumber stopped). The physical gamepad
reports still only arrive on the `xpad` node. So the config needs **two separate `source_devices`
entries**: one for the vendor node (worth keeping — force-feedback effect upload does succeed on it)
and one for the xpad node (the actual input). Dropping the xpad entry as "redundant now that the
vendor driver exposes a gamepad" leaves the virtual controller with no standard buttons at all,
while Steam happily picks the ungrabbed xpad node up as a *second* controller that does work — which
reads as "some buttons work, some don't" rather than as a config error. pastaq's 2026-09-02 report
that the gamepad enumerates on `*/input3` with the driver loaded did **not** reproduce here; on this
machine it is `*/input0` (xpad) and `*/input1` (vendor).

⚠️ **The two entries are told apart by name, not by `phys_path`.** The vendor entry's `*/input1`
pin was removed on 2026-09-09 (verified: composite device count, source list and every button
unchanged). `has_matching_evdev` (`src/config/mod.rs:855-861`) glob-matches the *whole* name string,
and the vendor entry's name alternatives — `ZOTAC Gaming Zone Gamepad` and the `hid-generic`-era
`Zotac Technology Limited ZOTAC GAMING ZONE` — cannot match the xpad node's plain `ZOTAC Gaming
Zone`, so the entries stay disjoint without the pin. The pin originally existed only to stop a
second composite device being spawned, which `unique: false` now handles; removing it also lets the
entry find the vendor node on hardware that enumerates it on another interface (the `*/input3`
report above). The xpad entry still pins `*/input0` — the same argument would allow dropping it, but
it has not been tested and standard buttons are what breaks if it goes wrong.

**Button → signal, current values.** The vendor driver hands out clean F16-F19 presses, one per
button, and the `hid-generic`-era oddities are gone — the physical HOME button no longer emits
`Meta+D` / `Ctrl+Alt+KP.` chords at all, so the `mapping_type: evdev: chord` entries that used to
carry it are dead weight and have been removed:

| Button | Keycode now | Keycode under `hid-generic` | Mapped to |
|---|---|---|---|
| ZOTAC | `KEY_F16` | `KEY_F17` | `Guide` |
| MORE / QAM | `KEY_F17` | `KEY_F18` | `QuickAccess` |
| HOME short | `KEY_F18` | `Meta`+`D` chord | `Screenshot` |
| HOME long | `KEY_F19` | `Ctrl`+`Alt`+`KP.` chord | `Guide` |
| Left paddle (M2) | `KEY_HOME` | same | `LeftPaddle1` |
| Right paddle (M1) | `KEY_END` | same | `RightPaddle1` |

This retroactively settles the whole PR #664 review argument: pastaq's proposed convention
(`F16→Guide`, `F17→QuickAccess`, `F18`/`F19` for HOME) was correct **for vendor-driver hardware**,
and the contradicting measurements from this machine were correct **for `hid-generic` hardware**.
Neither side was wrong; they were describing different kernels. Upstream's packaged `zone_type1.yaml`
uses exactly that convention. The local override keeps `F18→Screenshot` / `F19→Guide` instead of
upstream's `QuickAccess2`/`Keyboard` only because those two capabilities still produce no evdev
output on the `xbox-elite` target (`event_codes_from_capability` returns an empty vec; `xpad.rs`'s
`write_event` only special-cases `QuickAccess` and `Screenshot`).

**Why `QuickAccess2`/`Keyboard` do nothing here** (corrected 2026-09-09 — an earlier version of this
document said "no DBus target is attached", which is wrong):

- A DBus target **is** attached. The composite device's `DbusDevices` property lists
  `/org/shadowblip/InputPlumber/devices/target/dbus0`.
- `GamepadButton::QuickAccess2` and `GamepadButton::Keyboard` **do** have DBus translations —
  `src/input/event/dbus.rs:197-198` maps them to `Action::Quick2` (`"ui_quick2"`) and
  `Action::Keyboard`. So the mapping is not dead code in general, and pastaq's review claim that "it
  is not a no-op" was right in principle.
- What actually stops them is `InterceptMode`. `CompositeDevice::write_event`
  (`src/input/composite_device/mod.rs:1080-1105`) only forwards ordinary gamepad events to DBus
  targets when the intercept mode is `Always` or `GamepadOnly`; otherwise they go to the evdev
  targets alone. This device sits at `InterceptMode = 0` (none), so those two capabilities reach
  neither an evdev code nor a DBus signal.
- Unexplained: the 2026-08-27 hardware test ran with OpenGamepadUI actually running (both
  `opengamepadui --overlay-mode` and the `gamescope-session-ogui-steam` Gaming Mode session) and
  still saw no reaction, so something was not enabling intercept mode then either. Whatever the
  cause, on this machine's normal runtime state the practical conclusion stands — but state it as
  "intercept mode is off", not as "there is no DBus target".

**Paddles no longer need the hidraw workaround.** `KEY_HOME`/`KEY_END` arrive from the firmware
without anything writing a mapping first, and the kernel now exposes the remap knobs directly at
`/sys/.../1-4:1.3/0003:1EE9:1590.0003/btn_m{1,2}/remap` (plus `btn_a/remap`, `dpad_*/remap`, …), which
is the path `configure_via_sysfs()` was written for. `~/zotac-zone-tools/zotac-zone-paddles` and its
Steam shortcut are obsolete; the "re-run after power loss" instructions above no longer apply. The
scripts were moved to `~/zotac-zone-tools/obsolete/` on 2026-09-09.

**Dials: the physical dials work, but the configured dial mapping is a silent no-op** (measured
2026-09-09). Two independent paths leave the same rid=3 dial pulse, and only one survives:

- The vendor driver feeds rid=3 into its own `ZOTAC Gaming Zone Dials` evdev node as
  `REL_HWHEEL`/`REL_WHEEL` (`zotac-zone-hid-core.c:91-101`; the node declares exactly `EV=5`,
  `REL=140`). That node is a `group: mouse` source with `capability_map_id: zone1`, and
  `zone_type1.yaml` translates it to `LeftStickDial`/`RightStickDial` — **which then reaches
  nothing.** `TargetDeviceSet::write_event` (`src/input/composite_device/targets.rs:303-312`)
  routes by capability lookup and drops anything no attached target declares, logging one `trace`
  line. `xbox-elite` declares no relative axes at all (`xpad.rs:148-156`), and the mouse/keyboard
  targets declare only `Mouse::*`/`Keyboard::*`, so `Gamepad:Dial:*` is absent from the composite
  device's `TargetCapabilities` and every dial event dies there. Because the map consumes them,
  they don't reach the virtual mouse as plain `Mouse:Wheel` either.
- The same pulse *also* reaches the `ZOTAC Gaming Zone Keyboard` node as `KEY_VOLUMEUP` /
  `KEY_BRIGHTNESSUP`, which InputPlumber's keyboard target passes straight through. **That is what
  actually happens when you turn a dial: left = volume, right = screen brightness.**

Upstream's packaged config has the identical dead mapping (same `target_devices`, same rules), so
this is not a local misconfiguration. Don't "fix" the YAML — nothing is wrong with it; making dials
reach the gamepad would need a target that declares `Gamepad::Dial`, which `xbox-elite` will never
be. `/etc/inputplumber/capability_maps.d/zone_type1_dial.yaml` (id `zone1_dial`) is an unreferenced
leftover from the `hid-generic` era; no `source_devices` entry points at it.

A trap this exposes in general: a target module's `translate_event()` can look perfectly capable of
handling an event — the mouse target's is generic and its virtual device really does declare
`REL_WHEEL`/`REL_HWHEEL` — while the router never delivers it. When a mapping "does nothing", check
`TargetCapabilities` on the composite device over DBus **before** suspecting the map, the source, or
the hardware. Note also that `busctl monitor` on the InputPlumber service shows nothing useful: the
attached `dbus0` target only receives events in intercept mode, so a capture of a working button
press comes back empty and proves nothing.

The old standing warning still holds — that entry's name glob must never also match `ZOTAC Gaming
Zone Mouse`, or the touchpad's genuine scroll gets translated into dial events again.

**Why the `/etc` override still exists.** Because both it and the packaged config load
simultaneously (see "Config loading and overlay"), the override must stay a matching *superset* of
the packaged `source_devices` list or the uncovered device spawns a second composite device. It also
still carries three things the packaged config does not: the xpad (`*/input0`) entry, `unique: false`
on every entry, and the `Screenshot`/`Guide` targets for HOME. Removing `/etc/inputplumber` wholesale
would currently *lose* working behaviour, so the old "delete the overrides once the driver lands"
instruction is withdrawn.

**Known-bad states and how they present** (all seen during this session, worth recognising fast):

- Two `Zotac Zone` composite devices in `inputplumber devices list` → Steam shows two "Xbox Elite 2"
  controllers, buttons appear stuck/ghosting because two composite devices fight over the same
  sources. Cause is always coverage/`unique`, per "Config loading and overlay".
- One composite device, but the virtual controller has *only* the special buttons → the `*/input0`
  xpad source is missing or ungrabbed.
- A raw `ZOTAC Gaming Zone` controller visible in Steam alongside the virtual one → same thing; the
  xpad node isn't being grabbed.
- `inputplumber device N test` never shows a box for `Screenshot`, `QuickAccess`, `LeftPaddle1` even
  when they work, because the panel is built from each source's *declared* capabilities, not from
  capability-map output. `RightPaddle1/2` boxes *do* appear (the vendor gamepad node declares
  `BTN_TRIGGER_HAPPY5/6`) yet only light up via the keyboard path. Never read that panel as proof a
  mapping is broken.

**Upstream issue candidates found along the way** (none reported yet):

- `src/input/target/mod.rs` (~L551): a single `Err` from a target's `write_event`/`emit()` breaks the
  target's whole `run()` loop permanently, logging only at `debug` level, with no respawn — one bad
  event can silently kill a controller for the rest of the session.
- `unique` defaulting to `true` turns "an entry matched a second device" into "spawn a duplicate
  composite device", which is a surprising default and hard to diagnose from the logs.
- `filtered_events:` is present in every shipped capability-map YAML but is not a field on
  `CapabilityMapConfigV2` — silently ignored.
- Same-named configs in `/etc` and `/usr/share` are both loaded with no dedup, so an override that is
  a *subset* of the packaged config silently produces duplicate composite devices.
- The `REL_WHEEL`/`REL_HWHEEL` → single `Mouse::Wheel` collapse noted earlier in this document is
  still present in 0.79.0. This device finally has a real horizontal-axis source (the left dial),
  though its events are dropped before any target sees them, so the collapse stays latent here.
- A capability_map rule that translates into a capability **no attached target declares** is dropped
  by `TargetDeviceSet::write_event` (`src/input/composite_device/targets.rs:303-312`) with a single
  `trace` line — no warning, no config-load-time check. Upstream's own packaged `50-zotac-zone.yaml`
  ships exactly this (dial rules with no dial-capable target), so it is not a rare user mistake.
