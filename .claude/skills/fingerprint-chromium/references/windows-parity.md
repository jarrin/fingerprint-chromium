# Linux vs Windows Chrome: signal checklist

The question this file answers: what can still tell this Linux build apart from real Chrome on
Windows? It merges (a) what the patches actually do, read from the patch text, with (b) research
into what detectors check (CreepJS, FingerprintJS, BrowserScan, PixelScan, Cloudflare). Values
marked *(unverified)* come from memory or research, not measurement. Measure them against a real
Windows Chrome before you hardcode anything.

Detection weight: **H** is checked by most detectors, **M** by some, **L** is rare but conclusive.

## Covered (but check the caveats)

| Signal | Patch | Caveat |
|---|---|---|
| UA string, `navigator.platform` = `Win32` | 002 | The `Chrome/x` token is the real build version, while `fullVersionList` uses `kChromiumVersions[seed%4]`, so the two can disagree |
| `userAgentData` + `Sec-CH-UA*` headers | 002 | The brand list has **no `Google Chrome`** unless `--fingerprint-brand=Chrome`. GREASE and order are the same for every seed. `platformVersion` is always `19.0.0` (Win11 24H2), with no Win10 (`10.0.0`) option. `bitness`, `wow64` and `formFactors` are not set (they match on x86_64). The platform check in `UpdateUserAgentMetadataFingerprint` is case-sensitive |
| Worker / ServiceWorker navigator and SW fetch headers | 002 (partly) | Not verified. CreepJS compares the worker scope with the main thread (H) |
| hardwareConcurrency / deviceMemory | 005 / 021 | The `Device-Memory` client hint is not patched. A seed can give 26–32 cores, which is rare |
| Fonts | 006 | Only *hides* names, and maps Windows families to random Linux ones. Real Windows metrics, `system-ui` (Segoe UI), default families and emoji (Segoe UI Emoji vs Noto Color Emoji, H) need the actual font files plus fontconfig aliases |
| Canvas image / toDataURL / readPixels | 012 / 013 / 016 | Seeded LSB noise on ≤10 pixels hides the hash, but not the DirectWrite-vs-FreeType rendering class. `toBlob` probably gets no noise |
| measureText | 015 | **Broken**: the width comes out ≈ 0 (see patches.md) |
| Client rects | 014 | A constant offset per seed, trivial to cancel out |
| WebGL vendor/renderer + about 55 params + precision | 011 / 020 | Windows ANGLE D3D11 profiles. **No spoofing at all with `--fingerprint-platform=linux`**. The extension list is not spoofed, LOW/MEDIUM_INT precision is missing, and the extension-gated params answer before the extension is enabled |
| Audio | 003 | Noise on the OfflineAudioContext sample rate only. `AudioContext.baseLatency`, `outputLatency` and `sampleRate` are not covered |
| Timezone | 018 | Covered by `--timezone`; language by `--lang` / `--accept-lang` |
| Headless / webdriver / CDP | 010 / 009 / 001 | `fakeShadowRoot` (007) is itself a page-visible non-standard property |
| Hyphenation | upstream `bundle-hyphenation-patterns.patch` | Bundled, so ungoogled's usual "no hyphenation" tell is gone |

## Uncovered gaps, roughly by value

1. **`speechSynthesis.getVoices()`** (H). Windows lists `Microsoft David/Mark/Zira – English (United
   States)` plus about 19 network `Google …` voices. Linux lists speech-dispatcher/espeak voices or
   `[]`. See `content/browser/speech/tts_{win,linux}.cc`.
2. **WebGL extension list and caps vs the spoofed D3D11 renderer** (H). Check
   `getSupportedExtensions()` for WebGL1 and WebGL2. `WEBGL_debug_shaders` returns GLSL instead of
   HLSL (L, but conclusive). The fix is to capture and spoof the extension list per profile in 011.
3. **WebGPU** (M, rising). `requestAdapter()` is often `null` on Linux, or reports a Vulkan
   adapter. Windows reports D3D12, with `adapter.info` vendor/architecture and its own
   features/limits. There is no patch for this (`third_party/blink/renderer/modules/webgpu/`).
4. **Fonts and text rendering** (H). This is mostly an environment problem: ship or install the
   Windows fonts and alias them. CreepJS ties `Segoe Fluent Icons` to Win11, and `Segoe MDL2
   Assets`, `Bahnschrift` and `Ink Free` to Win10. It marks Linux on `Noto Color Emoji`/`Arimo`.
5. **Codecs / EME** (H/M).
   - `flags.gn` sets no `proprietary_codecs` / `ffmpeg_branding="Chrome"`, so
     `canPlayType('video/mp4; codecs="avc1…"')` is probably `""` where Windows says `probably`.
   - HEVC and `MediaCapabilities.decodingInfo().powerEfficient` differ without VA-API.
   - Widevine: `enable_widevine=true`, but with no component updater the CDM is missing, so
     `requestMediaKeySystemAccess('com.widevine.alpha')` rejects.
   - `RTCRtpSender.getCapabilities('video')` lacks H.264 unless the build has it.
6. **`navigator.share` / `canShare`** (M). They are present on Windows and absent on Linux. A
   per-platform `status` in `third_party/blink/renderer/platform/runtime_enabled_features.json5`
   decides this. Diff that file for other `"Win"`-only features.
7. **Screen and window metrics** (M–H). `screen.availHeight` on Windows is the height − 40 (Win10
   taskbar) or − 48 (Win11), with `availTop` 0. Xvfb gives avail == full size. The `outer − inner`
   frame deltas also differ, and so does `devicePixelRatio` (1/1.25/1.5 are common on Windows). The
   declared `--fingerprint-screen-*` / `-device-scale-factor` switches are **unimplemented**, a
   natural place to start.
8. **Audio context properties** (M). On Windows `sampleRate` is usually 48000 and `baseLatency` is
   about 0.01 (WASAPI) *(unverified)*. These come from PulseAudio/PipeWire on Linux, and headless
   servers report defaults.
9. **`mediaDevices.enumerateDevices()`** (M). Windows exposes `default` **and `communications`**
   device IDs. Servers/VMs may lack kinds entirely.
10. **System colours / CSS system fonts / `AccentColor`** (L–M). Windows palette vs Blink/GTK.
11. **Built-in AI APIs** (`Translator`, `LanguageDetector`, `Summarizer`) report `unavailable` in
    ungoogled vs `downloadable` in Chrome (L, rising) *(unverified)*.
12. **`navigator.keyboard.getLayoutMap()`**, the Battery API and `webkitSpeechRecognition`
    network errors (L).

## Equal on both, don't spend effort

- TLS JA3/JA4 and HTTP/2 fingerprint (same BoringSSL and `net/`), which you can verify with
  tls.peet.ws.
- V8 behaviour (error stacks, Math, Intl on x86_64).
- `performance.memory.jsHeapSizeLimit`.
- The PDF plugin list.
- `window.chrome.{app,csi,loadTimes}`.
- Shader precision on desktop GPUs.
- The architecture/bitness client hints on x86_64.
- Scrollbar width: Fluent scrollbars are the default on both *(unverified, measure it)*.

## Outside the browser

- **TCP/IP passive OS fingerprint** (p0f-style, M): Linux TTL is 64 and Windows is 128, and the
  SYN option order and window size differ too. A patch can't fix it. Use a proxy that terminates
  TCP, or `net.ipv4.ip_default_ttl=128`, which fixes only part of it.
- **Lie detection**: CreepJS checks `Function.prototype.toString`, descriptors and cross-realm
  consistency. C++ patches avoid most of this, but noise that changes between reads is itself a
  tell. Keep every spoofed value stable per seed.

## Sources

- CreepJS fonts module: https://github.com/abrahamjuliot/creepjs/blob/master/src/fonts/index.ts
- FingerprintJS sources: https://github.com/fingerprintjs/fingerprintjs/tree/master/src/sources
- Win11 detection via UA-CH: https://learn.microsoft.com/en-us/microsoft-edge/web-platform/how-to-detect-win11
- Per-platform Blink features: https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/runtime_enabled_features.json5
