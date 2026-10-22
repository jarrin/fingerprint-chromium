# fingerprint-chromium

*A fingerprint browser based on `Ungoogled Chromium`.*

> ⚠️ **Notice**: This project provides **no technical support**. Please do not contact me for help. Anyone who contacts you asking for payment is a **scammer** — do not be fooled.

## Build from Source

The full source (ungoogled-chromium plus the fingerprint patches in `patches/extra/fingerprint/`) lives on the `main` branch.

Build it by following the [`ungoogled-chromium` build documentation](https://github.com/ungoogled-software/ungoogled-chromium/blob/master/docs/building.md), using this repository in place of the `ungoogled-chromium` submodule.

## Features

### Fingerprint Support

You can use creepjs and browserleaks to test fingerprint modification effects.

* **User-Agent**: Modify the browser's `navigator.userAgent`, `navigator.platform`, `navigator.userAgentData` and `Client Hints` information.
* **Operating System**
* **Audio**
* **Plugins**: After removal in version 133+, the browser returns a fixed plugin list.
* **CPU Cores**
* **Memory**
* **WebGL Image**
* **WebGL Metadata**: Modify GPU vendor and graphics card model (currently only supports Linux).
* **Fonts**
* **Canvas Image**
* **Canvas Text**
* **ClientRects**
* **WebRTC**
* **Language Support**
* **Timezone Support**

### Automation Support
**Optimized for automation scenarios, offering the following features:**

1. **Fake Shadow DOM Support**  
   Adds the `fakeShadowRoot` property, equivalent to the `shadowRoot` property, enabling access to Closed Shadow Root for easier automation handling.

2. **Avoid CDP Detection**  
   Prevents Chrome DevTools Protocol (CDP) detection when invoking `Runtime.enable`, enhancing stealthiness for automation tools.

3. **Webdriver**  
   Sets `navigator.webdriver` to false to avoid detection of automation tools.

4. **Headless**  
   Only changes `User-Agent` from `HeadlessChrome` to `Chrome`. Other headless characteristics remain unchanged, use with caution.

### Enabling Fingerprint Features with Command Line Arguments

You can enable or customize fingerprinting and privacy protection features by passing command line arguments when launching the browser:

| **Command Line Argument** | **Description** | **Examples** |
|---------------------------|-----------------|------------------------------|
| **`--fingerprint`** | Specifies the fingerprint seed. When enabled, most fingerprinting features take effect | 32-bit integer |
| **`--fingerprint-platform`** | Specifies the operating system type | `windows`, `linux`, `macos` |
| **`--fingerprint-platform-version`** | Specifies the operating system version | Uses default version if not specified |
| **`--fingerprint-brand`** | Specifies the browser brand in `User-Agent` and `User-Agent Data` | Chrome, Edge, Opera, Vivaldi (default is Chromium if not specified) |
| **`--fingerprint-brand-version`** | Specifies the version number for the browser brand | Uses default version if not specified |
| **`--fingerprint-hardware-concurrency`** | Specifies the number of CPU cores | Integer value (randomly generated from fingerprint seed if not provided) |
| **`--disable-non-proxied-udp`** | Specifies WebRTC policy, by default non-proxied UDP connections are disabled | Recommended to keep default setting |
| **`--lang`** | Sets the browser language | Language code (e.g., `en-US`) |
| **`--accept-lang`** | Sets the languages accepted by the browser | Language codes (e.g., `en-US,en`) |
| **`--timezone`** | timezone | Timezone (e.g., `Asia/Shanghai`, `UTC`) |
| **`--proxy-server`** | proxy server| `http`, `socks` proxy (password authentication not supported) |
| **`--disable-spoofing`** | Disable specific fingerprint spoofing (Chrome 144+) | Comma-separated values: `font`, `audio`, `canvas`, `clientrects`, `gpu` |
| **`--fingerprint-gpu-vendor`** | ~~Customize GPU vendor for WebGL fingerprinting (Removed in Chrome 144)~~ | ~~Vendor string. Removed in Chrome 144~~ |
| **`--fingerprint-gpu-renderer`** | ~~Customize GPU renderer/model for WebGL fingerprinting (Removed in Chrome 144)~~ | ~~Renderer string. Removed in Chrome 144~~ |
| **`--disable-gpu-fingerprint`** | ~~Disable GPU fingerprint (Removed in Chrome 144, use `--disable-spoofing=gpu` instead)~~ | ~~Removed in Chrome 144~~ |

### **New Command Line Arguments in Chrome 139**

**GPU Fingerprinting Parameters**

> **Note**: `--fingerprint-gpu-vendor` and `--fingerprint-gpu-renderer` have been removed in Chrome 144. Use `--disable-spoofing=gpu` to disable GPU fingerprint spoofing instead.

Starting from Chrome 139, GPU fingerprinting is automatically enabled when using the `--fingerprint` parameter. The following optional parameters allow for custom GPU configuration:

- **`--fingerprint-gpu-vendor`** (Optional)
  - Customize the GPU vendor string for WebGL fingerprinting.
  - Examples: `Intel Inc.`, `NVIDIA Corporation`, `AMD`, `Apple`
  - If not specified, GPU vendor will be automatically generated based on the fingerprint seed.

- **`--fingerprint-gpu-renderer`** (Optional)
  - Customize the GPU renderer/model string for WebGL fingerprinting.
  - Examples: `Intel Iris OpenGL Engine`, `NVIDIA GeForce GTX 1060`, `AMD Radeon RX 580`
  - If not specified, GPU renderer will be automatically generated based on the fingerprint seed.

### **User-Agent Customization Command Line Arguments (Chrome 131)**

Chrome 131 introduced two command line arguments for advanced customization of `User-Agent` and `User-Agent Data`:

- **`--fingerprint-brand`**
  - Specifies the browser brand used in `User-Agent` and `User-Agent Data`.
  - Supported values: `Chrome`, `Edge`, `Opera`, `Vivaldi`, or custom brand names.
  - If not specified, the default brand is Chromium.

- **`--fingerprint-brand-version`**
  - Specifies the version number for the specified brand.
  - Default values: `Chrome`, `Edge`, `Opera`, `Vivaldi` all provide default versions, custom versions can also be passed.

These arguments enhance browser environment simulation capabilities, suitable for automation and testing scenarios. If `--fingerprint-brand` is not specified, the default brand will be used.

### **New Features in Chrome 142**

1. **Updated UserAgent and UserAgentData Fingerprint Implementation**
   - Improved fingerprint implementation for `navigator.userAgent` and `navigator.userAgentData`.

2. **Updated Audio Fingerprint Implementation**
   - Enhanced audio fingerprint modification. You can verify the effect at https://pixelscan.net/fingerprint-check

3. **Updated Canvas Fingerprint Implementation**
   - Improved Canvas fingerprint modification for better anti-detection.

4. **GPU Fingerprint Disable Option**
   - GPU fingerprint may have issues in some scenarios. Use `--disable-gpu-fingerprint` parameter to disable GPU fingerprint if needed.
   - ~~Removed in Chrome 144. Use `--disable-spoofing=gpu` instead.~~

### **New Features in Chrome 144**

1. **Removed GPU Fingerprint Parameters**
   - Removed `--fingerprint-gpu-vendor`, `--fingerprint-gpu-renderer`, and `--disable-gpu-fingerprint` parameters.

2. **New `--disable-spoofing` Parameter**
   - A comma-separated parameter to selectively disable specific fingerprint spoofing.
   - Available values: `font`, `audio`, `canvas`, `clientrects`, `gpu`
   - Example: `--disable-spoofing=font,gpu` disables font and GPU fingerprint spoofing.

## Usage Examples

Here are command line examples for several common use cases:

### Basic Usage

```bash
./chrome --fingerprint=1000 --user-data-dir=/tmp/chromium/1000 --timezone="America/Los_Angeles" --proxy-server="your_proxy_server_address"
```

### Customizing User-Agent

```bash
chrome.exe --fingerprint=2023 --fingerprint-platform=macos --fingerprint-platform-version="15.2.0" --fingerprint-brand="Edge"  --user-data-dir=%TEMP%\chromium
```

### Fingerprint Testing

| Testing Platform                                                                                            | Status                                                      |
| -------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| [**CreepJS**](https://abrahamjuliot.github.io/creepjs/)                                            | ✔️ 51.5%.                         |
| [**PixelScan**](https://pixelscan.net/fingerprint-check)                                           | ✔️ Pass audio fingerprint detection.                     |
| [**BrowserScan**](https://browserscan.net/)                                                        | ✔️ GPU fingerprint may have issues.                     |
| [**BrowserLeaks**](https://browserleaks.com/)                                                      | ✔️ Supports multiple fingerprint leak detection                                      |
| **Cloudflare**                                                                                     | ✔️                                                        |
| ‣ [Turnstile](https://nopecha.com/demo/turnstile)                                                  | ✔️                                                        |

## Credits

 * [Ungoogled Chromium](https://github.com/ungoogled-software/ungoogled-chromium)
 * [adryfish/fingerprint-chromium](https://github.com/adryfish/fingerprint-chromium), the original fingerprint-chromium by adryfish

## License

BSD-3-clause. See [LICENSE](LICENSE)