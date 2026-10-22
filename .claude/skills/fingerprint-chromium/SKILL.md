---
name: fingerprint-chromium
description: Orientation for the fingerprint-chromium repo, an ungoogled-chromium fork whose patches make a Chromium built on Linux pass as Google Chrome on Windows to fingerprinting and anti-bot scripts. Use it at the start of any task in this repo, and whenever the user asks to add, change, fix, debug or explain a fingerprint patch or a `--fingerprint*` switch, to spoof a browser signal (user agent, client hints, navigator.platform, fonts, canvas, WebGL/GPU, audio, client rects, timezone, hardware concurrency, device memory), to close a Linux-vs-Windows gap ("make it look more like Windows Chrome", "CreepJS/BrowserScan detects Linux"), to refresh patches with quilt, to build, or to rebase onto a new Chromium/ungoogled-chromium version, even when the user does not name the skill.
---

# fingerprint-chromium

The repo is **ungoogled-chromium** (Chromium version in `chromium_version.txt`) with a block of
fingerprint patches in `patches/extra/fingerprint/`, applied last in `patches/series`. There is no
Chromium source checkout here, only config and `.patch` files. Changing behaviour means editing a
patch's diff text, or regenerating it with quilt against a source tree on build-vm.

**Goal:** a Linux build, launched with `--fingerprint=<seed> --fingerprint-platform=windows`,
should look like stock Chrome on Windows 10/11 to every observable signal: JS APIs, HTTP Client
Hints, rendering and timing. Each profile is derived deterministically from the seed.

## Orient fast

1. Read the reference that matches the task. Each is a snapshot: verify a claim against the patch
   before relying on it.
   - `references/patches.md` covers each patch: the files it touches, the switches, how the seed is
     derived, and known bugs. Read it before touching any patch.
   - `references/windows-parity.md` lists the Linux-vs-Windows signals, which ones are covered and
     which are still gaps. Read it when the task is "look more like Windows".
   - `references/workflow.md` covers the repo layout, the build pipeline, quilt, validators, the
     rebase procedure and conventions. Read it before building, refreshing or adding a patch.
2. Look at the patch itself: `rg -n '<symbol>' patches/extra/fingerprint/`. The patches are
   git-style diffs. Their `+++ b/<path>` lines name the Chromium files they touch.
3. When the task needs upstream Chromium code that no patch touches, look it up at
   source.chromium.org for the version in `chromium_version.txt`. Don't guess.

## Rules of the road

- **Never build or download Chromium on this desktop.** Source trees, `ninja`, quilt refreshes and
  `devutils/validate_patches.py -l` run on **build-vm**. The local checks are
  `devutils/validate_config.py` and `devutils/check_all_code.sh`.
- **Edit patches as diffs, carefully.** When editing a `.patch` by hand, keep the hunk headers
  (`@@ -a,b +c,d @@`) consistent with the line counts. A miscounted hunk only fails at apply time on
  build-vm. Prefer a quilt refresh on build-vm for anything beyond a few lines.
- **Spoof consistently across layers.** A value exposed in several places has to change in all of
  them, or the mismatch is itself the tell:
  - JS on the main thread and in workers/service workers
  - the HTTP `Sec-CH-*` request headers
  - CSS media queries and rendering
  The classic example is `navigator.userAgentData` versus the `Sec-CH-UA-*` headers versus the UA
  string. 002 has to patch three sites for this reason.
- **Follow the existing seed and gating idioms.**
  - Read `kFingerprint` and derive the value with `std::hash<std::string>{}(seed + "<salt>")`.
  - Gate it on `--fingerprint` being present and on `--disable-spoofing` not containing the area.
  - A new switch goes in 000, both the constant and the renderer propagation list, and in the
    README switch table.
  - New patches are numbered from **022** and go at the end of `series`.
- **Don't break what's already spoofed.** Several patches edit hunks another patch added (for
  example 021 edits 005's hunk, and 012/015 edit bromite's). Changing an earlier patch shifts the
  context the later ones depend on.
- **Prefer real-device data.** Plausible-looking values are easy to detect. Spoofed values (GPU
  strings, WebGL limits, font lists, version tables) should come from captured real Windows Chrome
  profiles, and the hardcoded version tables in 002 should be bumped on every rebase.

## Testing a change

After a build on build-vm, run the binary with `--fingerprint=<n> --fingerprint-platform=windows
--fingerprint-brand=Chrome` and compare against a real Windows Chrome on CreepJS, BrowserScan,
PixelScan and browserleaks.com. Check the HTTP headers too, not only the JS values. Seed stability
matters: the same seed must give the same values across restarts, and different seeds must give
different profiles.
