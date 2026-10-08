MFOC is an open source implementation of "offline nested" attack by Nethemba.
Later was added so called "hardnested" attack by Carlo Meijer and Roel Verdult.

This program allow to recover authentication keys from MIFARE Classic card.

Please note MFOC is able to recover keys from target only if it have a known key: default one (hardcoded in MFOC) or custom one (user provided using command line).

This is a port to win32 x64 platform using native tools (Visual Studio 2019 + LLVM clang-cl toolchain).
This tree was also reworked for gnu toolchain (autotool + gcc like the original).

Based on the idea by vk496 to integrate mylazycracker into mfoc, forked from his tree.

For credits (there are many) just look at the AUTHORS file.

Uses
		libnfc 			https://github.com/nfc-tools/libnfc/
		libusb-1.0 		https://github.com/libusb/libusb
		pthreads4w		https://sourceforge.net/projects/pthreads4w/
		liblzma			https://tukaani.org/xz/

pthreads4w and liblzma are static linked.
All these libs are precompiled and included in src\lib

# Build from source
Windows:
Make sure you have Visual Studio 2019 with Desktop development with C++, C++ Clang Compiler for Windows and C++ Clang-cl for v142 build tools installed.
Open the solution and start compile.
The compiled zip package will be in dist.

Linux:
```
autoreconf -vis
./configure
make && sudo make install
```

# Usage #

`nfc.dll` and `libusb-1.0.dll` must be in the PATH (ideally in the same directory as the
executable). `nfc.dll` is linked against both the Windows PC/SC stack (`WinSCard.dll`) and
`libusb-1.0.dll`, so **`libusb-1.0.dll` is required even when you use a PC/SC reader.**

The libnfc build shipped here includes the `pcsc` / `acr122_pcsc`, `acr122_usb`, `pn532_uart`
(serial) and `arygon` drivers. Set up the driver matching your reader:

### ACR122U (PC/SC — recommended)
The ACR122U is driven through PC/SC. **No Zadig / libusbK needed.**
- Install the ACS PC/SC driver (or rely on the Windows inbox CCID driver) and make sure the
  **Smart Card** service (`SCardSvr`) is running.
- libnfc will auto-detect it via PC/SC. You can also pin it in `libnfc.conf`:
  `device.connstring = "pn532_usb"` is *not* used here — for the ACR122U use the PC/SC
  connstring, e.g. `device.connstring = "acr122_pcsc"`.

### PN532 over USB‑TTL (serial / UART)
A PN532 breakout on a USB‑TTL adapter is driven through `pn532_uart` over a virtual COM port.
- Install the **Silicon Labs CP210x VCP** driver so the adapter shows up as `COMx`.
- Tell libnfc which port to use, either in `libnfc.conf`:
  `device.connstring = "pn532_uart:COM3"` (replace `COM3` with your port),
  or via the `LIBNFC_DEFAULT_DEVICE` environment variable.
- To let libnfc probe serial ports automatically, enable the intrusive scan with `-I 1`
  (see options below).

### ACR122U in direct USB mode (alternative, optional)
Only if you use the `acr122_usb` driver instead of PC/SC: install a libusb-compatible driver
for the reader with Zadig (https://zadig.akeo.ie/) → Options → List All Devices → select your
reader → choose **WinUSB** (recommended for libusb-1.0) or **libusbK (v3.0.7.0)** → Replace Driver.
Then, in Device Manager, disable "Allow the computer to turn off this device to save power"
for the reader.

Put one MIFARE Classic tag that you want keys recovering.
Launching mfoc-hardnested, you will need to pass options, see
```
mfoc-hardnested -h
```

## Options

| Option      | Argument | Description |
|-------------|----------|-------------|
| `-h`        | *(none)* | print help and exit |
| `-C`        | *(none)* | skip testing default keys |
| `-F`        | *(none)* | force the hardnested keys extraction |
| `-Z`        | *(none)* | reduce memory usage |
| `-k key`    | hex key  | try the specified key in addition to the default keys |
| `-f file`   | path     | parse a file of keys to add to the default keys |
| `-P probnum`| number   | number of probes per sector (default 20) |
| `-T tol`    | number   | nonce tolerance half-range (default 20, i.e. 40 total) |
| `-O output` | path     | file in which the card contents will be written |
| `-I [0\|1]` | `0`/`1`  | **force libnfc intrusive scan**: `0` disables, `1` enables (sets `LIBNFC_INTRUSIVE_SCAN`). Useful to let libnfc probe serial ports for a PN532 UART reader. If omitted, libnfc's own default is used. |
| `-S`        | *(none)* | **static-nonce diagnostic** (see below) |

### `-S` — static-nonce diagnostic (added in this fork)
Selects the tag, collects several *plaintext* tag nonces (`nt`) and reports whether the nonce
is **constant**. A constant nonce is characteristic of a clone / "magic" chip (e.g. the Fudan
**FM11RF08S** family). This mode is **read-only** and **requires no key** — it does not attempt
any key recovery. It runs the test (block 0, 10 acquisitions) and exits; `-P`/`-T` do not affect
it. A "static nonce" verdict is a heuristic fingerprint, not a cryptographic proof.

#### Examples
```sh
# Static-nonce diagnostic only (no cracking): is this card a clone/magic chip?
mfoc-hardnested -S

# PN532 UART: let libnfc probe serial ports, then dump the card
mfoc-hardnested -I 1 -O mycard.mfd

# Force-disable intrusive scan (e.g. ACR122U PC/SC only)
mfoc-hardnested -I 0 -O mycard.mfd
```

## `detect_static_nonce()` — routine documentation

### Signature
```c
int detect_static_nonce(mftag t, mfreader r, uint8_t block, unsigned probes);
```

| Parameter | Meaning |
|-----------|---------|
| `t`       | Selected tag context (`mftag`). |
| `r`       | Reader context (`mfreader`). |
| `block`   | Block targeted by the AUTH-A command used to elicit a nonce (`-S` calls it with `0`). |
| `probes`  | Number of nonce acquisitions, clamped to `[2, 64]` (a value `< 2` becomes `5`, `> 64` becomes `64`). `-S` calls it with `10`. |

### Return value
| Value | Meaning |
|-------|---------|
| ` 1`  | **Static nonce** — `nt` identical across all acquisitions → likely clone / magic chip (e.g. FM11RF08S family). |
| ` 0`  | **Variable nonce** — `nt` varied (pseudo-random) → standard genuine chip. |
| `-1`  | **Inconclusive** — fewer than 2 nonces collected, or a device/property error occurred. |

### How it works
The first tag nonce (`nt`) of a MIFARE Classic authentication is sent by the card **in clear**,
before the Crypto-1 stream starts, so it can be read without any key. The routine:

1. Builds an `AUTH A` command for `block` (`MC_AUTH_A, block, 0x00, 0x00`) and appends the
   ISO 14443-A CRC.
2. Clamps `probes` to `[2, 64]`.
3. Repeats `probes` times:
   - **Re-selects the tag cleanly** (`mf_configure` + `mf_anticollision`) — a genuine chip only
     emits a fresh `nt` after a new anticollision.
   - Switches the reader to raw mode (disable `NP_HANDLE_CRC` and `NP_EASY_FRAMING`) to read the
     4-byte `nt` unmodified.
   - Transceives the AUTH command, captures the response, re-enables easy framing.
   - If fewer than 4 bytes are returned, logs a missed probe and continues; otherwise stores
     `nt = bytes_to_num(Rx, 4)`.
4. Restores normal reader config (`NP_HANDLE_CRC` + `NP_HANDLE_PARITY` on).
5. If fewer than 2 nonces were collected → returns `-1`.
6. Compares every collected nonce to the first one: all identical → static nonce warning,
   return `1`; any difference → "variable nonce / standard chip", return `0`.

### Limitations & notes
- **Heuristic, not proof.** A constant `nt` strongly suggests a clone/magic or special chip, but
  reader/antenna or timing conditions can also perturb acquisition.
- **Fixed parameters under `-S`:** always `block = 0`, `probes = 10`; `-P`/`-T` have no effect.
- **Up to 64 nonces** are stored (internal buffer); the clamp keeps `probes` within that bound.
- **Reader state:** the routine toggles `NP_HANDLE_CRC` / `NP_EASY_FRAMING` / `NP_HANDLE_PARITY`
  and restores CRC/parity on exit; under `-S` the program exits immediately afterwards.

---

Inspired by the static-nonce / backdoor research published by Philippe Teuwen (Quarkslab, 2024),
mirroring the kind of fingerprint reported by Proxmark Iceman's `hf mf info`.
