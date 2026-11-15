# Waveshare ESP32-S3-Touch-AMOLED-1.75C: Project Knowledge Base

Last researched: 2026-11-15. Nothing here has been built or run on hardware yet.

Every fact carries a status:

- **[V]** verified against a primary source (vendor page, vendor wiki, Espressif docs or registry)
- **[S]** taken from the sibling board (non-C `ESP32-S3-Touch-AMOLED-1.75`) or a third party; likely but unconfirmed for the 1.75C
- **[A]** assumption or general ESP-IDF knowledge; confirm before relying on it

---

## 1. Decisions

| Topic | Decision | Why |
|---|---|---|
| Framework | ESP-IDF (FreeRTOS) | Only framework with a vendor BSP for this board. Wi-Fi and BLE stacks need an RTOS. |
| Language | C for drivers and board glue, C++ allowed for application code | BSP, LVGL and all chip drivers are C. ESP-IDF builds C and C++ side by side. |
| UI | LVGL via the vendor BSP | The BSP already brings it in. |
| ESP-IDF version | v5.5.x, pinned | A third-party project built and measured on this exact board with v5.5.4. Re-check the BSP before moving to v6.x. |
| Host | macOS, EIM via Homebrew | Espressif's documented install route. |
| Editor | VS Code + ESP-IDF extension (optional) | Waveshare's recommendation; `idf.py` on the command line is enough. |
| Build flow | ESP-IDF's native CMake build, driven by `idf.py` | See section 1.1. |

### 1.1 Build flow

**Decision: the stock ESP-IDF CMake + Ninja build via `idf.py`, with the IDF version pinned in two places: EIM locally and the `espressif/idf:vX.Y.Z` Docker image in CI.**

What the ESP-IDF build does beyond compiling, and what any replacement would have to rebuild:

- Kconfig / menuconfig → `sdkconfig` → generated `sdkconfig.h` [V]
- IDF Component Manager: resolves `idf_component.yml` (the BSP is delivered this way) and writes `dependencies.lock` [V]
- linker fragment generation (ldgen) for IRAM/flash placement [A]
- separate bootloader sub-build, partition table generation, esptool flash arguments [A]

The vendor BSP, the vendor examples and the a3d reference project all ship as ESP-IDF CMake components or projects [V]. Using the native build means they work unchanged.

Pinning and reproducibility:

| Layer | Pin | Status |
|---|---|---|
| ESP-IDF and toolchain, local | `eim install -i v5.5.x` | [V] |
| ESP-IDF and toolchain, CI | `espressif/idf:v5.5.x` Docker tag. Docker Hub has `v5.5` through `v5.5.6` and `release-v5.5` (checked 2026-11-15) | [V] |
| Components (BSP, LVGL, drivers) | `dependencies.lock`, committed | [V] |
| Config | `sdkconfig.defaults`, committed | [V] |

CI: `espressif/esp-idf-ci-action@v1` with `esp_idf_version: v5.5.x` and `target: esp32s3`. Its `esp_idf_version` input takes Docker Hub tags [V]. Local and CI should use the same patch version. v5.5.4 is the version the a3d project measured on this board; v5.5.6 is the newest 5.5.x tag on Docker Hub.

Host-side testing: ESP-IDF's `linux` target (`idf.py --preview set-target linux`) runs logic tests on the host. It is experimental in v5.5 and has only mock support for `driver` [V], so use it for application logic, not drivers. To iterate on the UI, a separate plain-CMake LVGL + SDL simulator build is an option [A].

An optional `Makefile` or `justfile` can wrap common `idf.py` commands. It must not replace them.

#### Rejected build flows

| Option | Reason |
|---|---|
| Bazel | No maintained ESP-IDF rules found. `simonhorlick/bazel_esp32` targets Arduino and is about eight years old. The one current Bazel-based ESP32-S3 project found (GizClaw/gizos) keeps ESP-IDF's CMake components and an external IDF checkout; Bazel orchestrates and does not replace the IDF build [V]. Bazel would have to either shell out to `idf.py` (adding a layer for no gain) or reimplement Kconfig, ldgen, the bootloader build and the component manager. Revisit only if this becomes part of a larger Bazel monorepo. |
| PlatformIO (`espidf` framework) | Works, but Espressif states it does not maintain ESP-IDF support in PlatformIO [V]. Official IDF support lags upstream: platform 6.12.0 added 5.5 and 6.13.0 added 5.5.3 [V]. It adds a second config layer (`platformio.ini`) on top of sdkconfig and the component manager. |
| Plain CMake without `idf.py` | Possible because `idf.py` is a wrapper around CMake [V], but you lose `flash`, `monitor`, `menuconfig` and component manager integration for no gain. |
| Meson, Make, SCons | No ESP-IDF support. The GNU Make build was removed after v4.x [A]. |

### Rejected options

| Option | Reason |
|---|---|
| Zephyr | No upstream board definition or CO5300 display driver found. Would mean writing display, touch and devicetree support from scratch. Revisit only if cross-vendor portability becomes a requirement. |
| Bare metal (no RTOS) | Gives up Wi-Fi and BLE in practice, and the board runs display, touch, audio and network concurrently. |
| Arduino | Officially supported, but it wraps ESP-IDF with less control over sdkconfig, partitions and components. Fine for a quick hardware smoke test. |
| Rust (`esp-hal`, no_std) | `esp-hal` reached 1.0 in October 2025, but the ESP32-S3 is Xtensa and needs the forked compiler installed with `espup`. The 1.0 stable surface is small (init, GPIO, UART and a few core drivers); the rest sits behind an `unstable` feature. No BSP for this board, so CO5300, CST9217, AXP2101 and the audio chain would be your own work. |
| Rust (`esp-idf-hal`, std) | Workable, but you would wrap the C BSP through FFI for little gain. |
| ESPHome | A community config exists for the non-C sibling. Good for Home Assistant gadgets, not for custom firmware. |

---

## 2. Hardware

### 2.1 Bill of materials

| Function | Part | Bus | Status |
|---|---|---|---|
| SoC | ESP32-S3R8, dual-core Xtensa LX7, 240 MHz | n/a | [V] |
| RAM | 512 KB SRAM + 8 MB PSRAM (in-package) | n/a | [V] |
| Flash | 32 MB NOR | SPI | [V] |
| Display | 1.75" round AMOLED, 466 × 466, 16.7M colours, driver CO5300 | QSPI | [V] |
| Touch | CST9217 capacitive | I2C | [V] |
| Audio codec (playback) | ES8311 | I2C + I2S | [V] |
| Audio ADC (mics) | ES7210, dual-mic array | I2C + I2S | [V] |
| Speaker | built in | via ES8311 | [V] |
| PMIC | AXP2101, charging and battery management | I2C | [V] |
| Battery | 3.7 V Li-ion, MX1.25 2-pin header | n/a | [V] |
| IMU | QMI8658, 6-axis | I2C | [V] |
| Radio | 2.4 GHz Wi-Fi b/g/n, Bluetooth 5 LE, onboard antenna | n/a | [V] |
| USB | Type-C to the ESP32-S3's own USB (flash + logs) | n/a | [V] |
| Buttons | PWR (programmable), BOOT | GPIO | [V] |
| Expansion | I2C, UART, USB and GPIO as solder pads (no header) | n/a | [V] |
| Case | CNC aluminium | n/a | [V] |

### 2.2 Open hardware questions

Resolve these from the schematic (link in section 9) and record the answers here.

| Question | What the sources say |
|---|---|
| Is there an RTC chip? | Product comparison table marks RTC as absent on the 1.75C. Wiki intro says the board "integrates" an RTC but names no chip. The non-C sibling has a PCF85063. |
| Is there a TF (microSD) slot? | Comparison table marks TF as absent on the 1.75C. Wiki does not mention one. The BSP still pulls in FATFS. |
| Which variant do I own? | SKUs include standard, with-battery and GPS (LC76G) versions. Note yours here: `________` |
| I2C addresses of each chip | Not published on the wiki. Run an I2C scan and record. |

### 2.3 Pin map

**Not verified for the 1.75C.** The values below are from the non-C sibling [S]. Replace them with values from the BSP header or schematic, then mark the table [V].

| Signal | GPIO (sibling) | 1.75C confirmed |
|---|---|---|
| I2C SDA (shared bus) | 15 | |
| I2C SCL (shared bus) | 14 | |
| Touch INT | 11 | |
| Touch RST | 40 | |
| I2S LRCLK | 45 | |
| I2S BCLK | | |
| I2S MCLK | | |
| I2S DOUT (to ES8311) | | |
| I2S DIN (from ES7210) | | |
| LCD QSPI CS | | |
| LCD QSPI SCLK | | |
| LCD QSPI D0..D3 | | |
| LCD RST | | |
| LCD TE | | |
| PA enable | | |
| PWR button | | |
| BOOT button | 0 [A] | |

On Waveshare's other CO5300 boards the display has a dedicated QSPI bus and the I2C bus is shared by touch, PMIC, IMU and codecs [S]. Expect the same here.

---

## 3. Software stack

```
application (C / C++)
        |
      LVGL
        |
vendor BSP: waveshare/esp32_s3_touch_amoled_1_75c
        |
esp_lcd (CO5300 QSPI) | touch (CST9217) | esp_codec_dev (ES8311, ES7210) | AXP2101 | QMI8658
        |
ESP-IDF v5.5.x  (FreeRTOS, Wi-Fi, BLE, drivers)
        |
ESP32-S3R8
```

### 3.1 Vendor BSP

- Registry name: `waveshare/esp32_s3_touch_amoled_1_75c`, version 3.0.0, target `esp32s3` [V]
- Source: `github.com/waveshareteam/Waveshare-ESP32-components`, path `bsp/esp32_s3_touch_amoled_1_75c` [V]
- Has 8 dependencies; reported to include LVGL, the ES7210 codec driver, SPIFFS, FATFS and USB [S]
- API is expected to follow the esp-bsp convention (`bsp_display_start()`, `bsp_display_lock()`, `bsp_display_unlock()`, `bsp_i2c_init()`, audio init helpers) [A]. Confirm in the BSP header.
- The pin definitions for section 2.3 live in the BSP's main header.

### 3.2 Vendor examples

- Repo: `github.com/waveshareteam/ESP32-S3-Touch-AMOLED-1.75C` with ESP-IDF and Arduino examples plus firmware releases [V]
- Use the example `sdkconfig` and partition table as the reference for flash and PSRAM settings.

---

## 4. macOS toolchain

### 4.1 One-time setup

```bash
# prerequisites
brew install libgcrypt glib pixman sdl2 libslirp dfu-util cmake python

# Espressif Installation Manager (CLI)
brew tap espressif/eim
brew install eim

# pinned ESP-IDF
eim install -i v5.5.4
```

EIM prints how to activate the environment when it finishes. Record the exact activation command here once known: `________`

Python 3.10 is the minimum for ESP-IDF [V].

### 4.2 Project bootstrap

```bash
idf.py create-project amoled175c-app
cd amoled175c-app
idf.py set-target esp32s3
idf.py add-dependency "waveshare/esp32_s3_touch_amoled_1_75c^3.0.0"
idf.py build
```

### 4.3 Flash and monitor

```bash
ls /dev/cu.usbmodem*                 # find the port
idf.py -p /dev/cu.usbmodemXXXX flash monitor
```

- Exit the monitor with `Ctrl-]`.
- If the board does not enumerate: hold BOOT, tap PWR or replug, release BOOT. This forces download mode [A].
- The USB port is the SoC's native USB, so no CH34x or CP210x driver should be needed [A].

### 4.4 Debugging

The ESP32-S3 has a built-in USB Serial/JTAG controller. If the Type-C port is wired to it, `idf.py openocd` and `idf.py gdb` work over the same cable with no probe [A]. Confirm against the schematic.

---

## 5. Repository layout (proposed)

```
.
├── CMakeLists.txt
├── sdkconfig.defaults          # committed
├── sdkconfig                   # NOT committed
├── partitions.csv
├── dependencies.lock           # committed, pins component versions
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml       # BSP dependency
│   └── main.c
├── components/                 # own drivers and modules
├── docs/
│   └── KNOWLEDGE_BASE.md       # this file
└── .gitignore
```

`.gitignore`:

```
build/
managed_components/
sdkconfig
sdkconfig.old
.vscode/
*.pyc
```

`main/idf_component.yml`:

```yaml
dependencies:
  idf: ">=5.5,<6.0"
  waveshare/esp32_s3_touch_amoled_1_75c: "^3.0.0"
```

### 5.1 `sdkconfig.defaults` starting point

All [A]. Diff against the vendor example's `sdkconfig` before trusting it.

```
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y

# 8 MB octal PSRAM inside the S3R8
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y

# 32 MB flash
CONFIG_ESPTOOLPY_FLASHSIZE_32MB=y

# logs over native USB
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y

CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
```

**Risk: 32 MB flash.** Flash above 16 MB needs 32-bit addressing. Support for that on quad flash with the ESP32-S3 has been restricted in ESP-IDF [A]. Copy whatever the vendor example does. If it is troublesome, configure 16 MB and leave the upper half unused until it is needed.

---

## 6. Language guidance

- **C** for anything touching ESP-IDF APIs, LVGL or the BSP. Matches every upstream example, so snippets paste in unchanged.
- **C++** is fine for application logic (state machines, UI screens). Keep exceptions and RTTI off unless needed; wrap C headers in `extern "C"`.
- **MicroPython / CircuitPython**: no board port found. Not researched further.
- **Rust**: see rejected options in section 1. Revisit if a community BSP for this board appears.

---

## 7. Known gotchas

| Issue | Detail | Status |
|---|---|---|
| Column offset | Visible area starts at column 6. A custom panel driver must call `esp_lcd_panel_set_gap(panel, 6, 0)` or the image is shifted and wraps. The BSP's init table handles this. | [S] |
| Byte order | The BSP declares `BSP_LCD_BIGENDIAN 0`, but one third-party author found the panel reads big-endian (wrong colours otherwise). Test with pure red, green and blue fills. | [S] |
| Round panel | 466 × 466 framebuffer, circular glass. Corners are not visible; keep UI inside the inscribed circle. | [V] |
| Framebuffer size | Full frame at RGB565 is 466 × 466 × 2 ≈ 434 KB. That does not fit comfortably in internal SRAM; allocate LVGL buffers from PSRAM or use partial buffers. | [A] |
| Shared I2C bus | Touch, PMIC, IMU and both codecs likely share one bus. Initialise it once and guard access. | [S] |
| AXP2101 rails | The PMIC powers the peripherals. Display or audio may stay dark until the correct rails are enabled. The BSP should do this; a custom init must too. | [A] |
| AMOLED burn-in | Avoid static full-brightness content; dim or sleep the panel when idle. | [A] |
| BSP weight | The BSP drags in LVGL, codecs and filesystems. For a minimal image, lift only the panel init from it (Apache-2.0 per the third-party report). | [S] |

---

## 8. Bring-up checklist

- [ ] Flash the vendor factory image or a stock example; confirm display, touch and audio work
- [ ] Download the schematic and fill in section 2.3
- [ ] Resolve the RTC and TF questions in section 2.2
- [ ] I2C scan; record addresses
- [ ] Build an empty project with the BSP on v5.5.4
- [ ] Colour test: red, green, blue, white fills
- [ ] Touch coordinates match display orientation
- [ ] Speaker tone, then mic loopback
- [ ] Battery voltage and charge state readable from AXP2101
- [ ] Wi-Fi connects while the display is refreshing
- [ ] Measure idle and sleep current on battery
- [ ] Record the activation command in section 4.1

---

## 9. References

Vendor
- Product page: https://www.waveshare.com/esp32-s3-touch-amoled-1.75c.htm
- Wiki: https://docs.waveshare.com/ESP32-S3-Touch-AMOLED-1.75C
- Resources index: https://docs.waveshare.com/ESP32-S3-Touch-AMOLED-1.75C/Resources-And-Documents
- Examples repo: https://github.com/waveshareteam/ESP32-S3-Touch-AMOLED-1.75C
- Schematic: https://github.com/waveshareteam/ESP32-S3-Touch-AMOLED-1.75C/raw/main/hardware/schematic/ESP32-S3-Touch-AMOLED-1.75C-schematic.pdf
- BSP (registry): https://components.espressif.com/components/waveshare/esp32_s3_touch_amoled_1_75c
- BSP (source): https://github.com/waveshareteam/Waveshare-ESP32-components

Datasheets
- ESP32-S3: https://documentation.espressif.com/esp32-s3_datasheet_en.pdf
- ESP32-S3 TRM: https://documentation.espressif.com/esp32-s3_technical_reference_manual_en.pdf
- QMI8658: https://files.waveshare.com/wiki/common/QMI8658C.pdf
- AXP2101: https://files.waveshare.com/wiki/common/X-power-AXP2101_SWcharge_V1.0.pdf
- ES8311: https://files.waveshare.com/wiki/common/ES8311.DS.pdf
- CO5300, CST9217, ES7210: not linked by Waveshare; source separately

Espressif
- macOS setup: https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/get-started/macos-setup.html
- esp-hal (Rust): https://docs.rs/crate/esp-hal/latest

Third party
- a3d example on this board (gap, byte order, IDF v5.5.4): https://components.espressif.com/components/0015/a3d/versions/0.9.3/examples/ESP-IDF/Waveshare_ESP32-S3-Touch-AMOLED-1.75C
- ESPHome config for the non-C 1.75 (sibling pins): https://devices.esphome.io/devices/waveshare-esp32-s3-touch-amoled-1.75
