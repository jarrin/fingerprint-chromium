# Fingerprint patches: what each one does

Paths are Chromium-relative unless they start with `patches/`. The patches live in
`patches/extra/fingerprint/` and are the last 18 entries of `patches/series` (lines 112–129).
Snapshot of Chromium 148.0.7778.215-1: re-check line numbers and values after a rebase.

## Shared infrastructure

- **Switches** are defined in `components/ungoogled/ungoogled_switches.{cc,h}` (namespace
  `switches`, added by 000) and forwarded to renderers in
  `RenderProcessHostImpl::PropagateBrowserCommandLineToRenderer`
  (`content/browser/renderer_host/render_process_host_impl.cc`). A new switch needs both: the
  constant, and an entry in that propagation list, or the renderer never sees it.
- **Data tables**: `components/ungoogled/fingerprint_data.h` (002: brands, versions, platforms)
  and `components/ungoogled/webgl_config_data.h` (011: GPU profiles), both in the
  `components/ungoogled/BUILD.gn` component.
- **There is no shared seed accessor.** Each call site reads
  `base::CommandLine::ForCurrentProcess()->GetSwitchValueASCII(switches::kFingerprint)` itself and
  parses it one of three ways:
  - `std::stoull` cast to `uint32_t` (005, 021): **throws on a non-numeric seed**.
  - `base::StringToInt` / `StringToUint64` (002, 011): quietly yields 0.
  - `std::hash<std::string>{}(seed + salt)` (003, 006, 012–015). This is libc++'s hash, stable
    per build but not across toolchains.
- **`--disable-spoofing`** is a copy-pasted substring check, not a comma split:
  `!HasSwitch(kDisableSpoofing) || value.find("<area>") == npos`. Areas: `audio` (003),
  `font` (006), `canvas` (012, 013, 015, 016), `clientrects` (014), `gpu` (011, 020).
- **Platform comparisons are inconsistent.** Most call sites lowercase `--fingerprint-platform`,
  but `UpdateUserAgentMetadataFingerprint` (002) and 006 do not. Always pass lowercase `windows`.

## Switch table (000)

| Constant | Switch | Read by |
|---|---|---|
| `kFingerprint` | `fingerprint` | everything (the seed) |
| `kFingerprintBrand` / `kFingerprintBrandVersion` | `fingerprint-brand` / `-brand-version` | 002 |
| `kFingerprintPlatform` / `kFingerprintPlatformVersion` | `fingerprint-platform` / `-platform-version` | 002, 006, 011 |
| `kFingerprintHardwareConcurrency` | `fingerprint-hardware-concurrency` | 005 |
| `kDisableSpoofing` | `disable-spoofing` | 003, 006, 011–016, 020 |
| `kFingerprintTimezone` | **`timezone`** (no prefix) | 018 |
| `kFingerprintScreenWidth/Height`, `kFingerprintDeviceScaleFactor`, `kFingerprintLocation` | `fingerprint-screen-*`, `-device-scale-factor`, `-location` | **nothing**: declared and propagated only |

## Per patch

- **000-add-fingerprint-switches**: the switches above, plus their propagation to renderers.
- **001-disable-runtime.enable** (`v8/src/inspector/v8-runtime-agent-impl.{cc,h}`): hides CDP
  `Runtime.enable` side effects. `addBindings()` returns early, `messageAdded()` stops
  reporting, and `enabled()` always returns false. It is unconditional, so CDP clients lose console
  messages and `Runtime.addBinding`.
- **002-user-agent-fingerprint**: the largest behavioural patch.
  - Files: `components/embedder_support/user_agent_utils.cc`,
    `third_party/blink/common/user_agent/user_agent_metadata.cc` (+ public `.h`),
    `content/browser/client_hints/client_hints.cc`,
    `third_party/blink/renderer/core/{frame/navigator.cc, frame/navigator_ua.cc,
    execution_context/navigator_base.cc, loader/frame_fetch_context.cc}`.
  - UA string: drops the "Headless" prefix. For `windows` the platform token is
    `Windows NT 10.0; Win64; x64`. The brand suffix (` Edg/…`, ` OPR/…`, ` Vivaldi/…`) comes
    from `GetUserAgentFingerprintBrandInfo()`. The `Chrome/x` token is still the real build
    version.
  - `UpdateUserAgentMetadataFingerprint()` overrides `navigator.userAgentData`, the navigation
    `Sec-CH-UA*` headers (`client_hints.cc`) and subresource hints (`frame_fetch_context.cc`).
  - Windows values:
    - `navigator.platform` returns `Win32` and `userAgentData.platform` returns `Windows`.
    - `platformVersion` is `--fingerprint-platform-version`, otherwise `kWindowsVersions[seed % 1]`,
      which is always `19.0.0`.
    - `architecture` is set to `x86`. `bitness`, `wow64`, `model` and `formFactors` are not set.
  - Versions:
    - `kChromiumVersions[seed % 4]` (148.0.7778.x).
    - `kEdgeVersions[seed % 4]`.
    - The GREASE brand and shuffle order are seeded by the **major version**, not the seed, so
      they are identical for every profile.
    - With no `--fingerprint-brand`, the brand list is GREASE + `Chromium` only, with no
      `Google Chrome`.
- **003-audio-fingerprint** (`modules/webaudio/offline_audio_context.cc`): adds seeded noise of
  ±0.01 to the `OfflineAudioContext` sample rate. `getNoiseData()` is dead code.
- **005-hardware-concurrency** (`core/frame/navigator_concurrent_hardware.cc`):
  - The switch wins.
  - Otherwise the value is `((seed % 13) + 4) * 2`, which gives even numbers from 8 to 32.
- **006-font-fingerprint** (`platform/fonts/font_cache.cc`, `FontCache::GetFontPlatformData`):
  - Holds hardcoded font lists per platform (`GetPlatformFonts`).
  - When spoofing Windows on Linux, a requested Windows-list family is mapped to
    `linux_fonts[hash % n]`, so it renders with Linux metrics. Any other family is hidden 95% of the
    time.
  - The `IsBasicFont()` allowlist (Arial, Calibri, Segoe UI, Times New Roman, Courier New, …)
    passes through to host fontconfig.
- **007-shadow-root**: adds `Element.prototype.fakeShadowRoot`, which exposes closed shadow
  roots. It is unconditional and visible to pages (detectable).
- **009-webdriver**: `navigator.webdriver` no longer becomes true from `--enable-automation`.
- **010-headless**: renames the headless product from `HeadlessChrome` to `Chrome`.
- **011-gpu-info**: new `modules/webgl/webgl_fingerprint.{h,cc}` and
  `components/ungoogled/webgl_config_data.h`. The data header was auto-generated by a
  `gen_webgl_config.py` that is **not in the repo**.
  - Profile selection is in `GetActiveWebGLConfig()`:
    - `windows` (the default when no platform is given) picks from 13 ANGLE/D3D11 profiles.
      They are mostly Intel Iris Xe/UHD/Arc, plus an NVIDIA RTX 3060/4060 Laptop and an AMD
      Radeon.
    - `macos` picks from 2 Apple M profiles.
    - **Any other value, including `linux`, returns nullptr, so nothing is spoofed.**
    - The index is `seed % size`, and the profile is not cached.
  - The hooks spoof `getParameter` for about 55 limits, UNMASKED_VENDOR/RENDERER,
    `getShaderPrecisionFormat` and the `antialias` context attribute.
  - It does not spoof `getSupportedExtensions`/`getExtension`, LOW_INT/MEDIUM_INT precision, or
    WebGPU.
  - It answers the extension-gated params (MAX_TEXTURE_MAX_ANISOTROPY_EXT,
    MAX_DRAW_BUFFERS_WEBGL) even before the extension is enabled, which is detectable.
- **012-canvas-get-image-data** (`canvas2d/base_rendering_context_2d.cc`,
  `platform/graphics/{image_data_buffer.cc, static_bitmap_image.cc}`):
  - Rewrites bromite's `ShuffleSubchannelColorData` to be seeded.
  - It flips the LSB of R/G/B on up to 10 "edge" pixels.
  - It also adds noise in `ImageDataBuffer::EncodeImage`.
  - It ignores `srcX/srcY`, so overlapping `getImageData` reads disagree.
- **013-canvas-toDataURL** (`core/html/canvas/html_canvas_element.cc`): shuffles a copy of the
  snapshot before encoding.
- **014-client-rects** (`core/dom/{document,element,range}.cc`, `ui/gfx/geometry/quad_f.*`):
  - Replaces bromite's scale with a constant seeded offset of ±0.001 px, applied to
    `getClientRects` and `getBoundingClientRect` (Element and Range).
  - Widths and differences between rects are unchanged.
- **015-canvas-measure-text**: **bug**. It passes `noise_x` (about ±5e-6) to bromite's
  `TextMetrics::Shuffle(factor)`, which *multiplies* the metrics, so `measureText().width` comes
  out ≈ 0. The fix is to pass `1 + noise_x`.
- **016-webgl-readPixels** (`webgl_rendering_context_base.cc`, `ReadPixelsHelper`):
  - Shuffles the read buffer.
  - GL_RGB is a no-op.
  - Non-UNSIGNED_BYTE types are treated as RGBA8, which risks corrupting float data.
- **018-timezone** (`core/timezone/timezone_controller.cc`): `--timezone=<IANA>` overrides the
  ICU zone. It also ignores CDP `Emulation.setTimezoneOverride`.
- **020-webgl-getparameter-timing**: adds a 16 µs busy-wait when returning spoofed UNMASKED_*
  values, to copy native timing.
- **021-device-memory-cap** (`core/frame/navigator_device_memory.cc`): edits a hunk 005 added.
  `deviceMemory` becomes `{8,16,32}[seed % 3]`. The `Device-Memory` client hint is not patched.

## Bromite base patches and how they interact

- `patches/extra/bromite/fingerprinting-flags-client-rects-and-measuretext.patch` (series line 52)
  adds `TextMetrics::Shuffle` and `Document::GetNoiseFactorX/Y`.
  - 014 and 015 take over most of its hooks.
  - It does not take over `quad.Scale(...)` in `Element::GetClientRectsNoAdjustment`, which still
    runs behind the bromite flag.
- `patches/extra/bromite/flag-fingerprinting-canvas-image-data-noise.patch` (series line 54) owns
  the `ImageEncoder::Encode` hook.
  - That hook is still gated by the bromite flag, so `toBlob`/`convertToBlob` probably get no noise
    from `--fingerprint` alone.
- `patches/extra/ungoogled-chromium/add-flag-to-spoof-webgl-renderer-info.patch` (`kSpoofWebGLInfo`)
  is overridden by 011.
