# mfoc-hardnested — fork `xavave/mfoc-hardnested-and-static-nonce`

Documentation for the options added in this fork: `-S` (static-nonce diagnostic)
and `-I` (force intrusive scan), plus the `detect_static_nonce()` routine.

---

## 1. README section (ready to paste)

### Additional options in this fork

This fork adds two command-line options on top of upstream
`nfc-tools/mfoc-hardnested`:

| Option      | Argument | Description |
|-------------|----------|-------------|
| `-S`        | *(none)* | **Static-nonce diagnostic.** Selects the tag, collects several *plaintext* tag nonces (`nt`) and reports whether the nonce is **constant**. A constant nonce is characteristic of a clone / "magic" chip (e.g. the Fudan **FM11RF08S** family). Read-only, **no key required**, does not attempt key recovery. Runs the test and exits. |
| `-I [0\|1]` | `0` or `1` | **Force libnfc intrusive scan.** `-I 0` disables it, `-I 1` enables it. Sets the `LIBNFC_INTRUSIVE_SCAN` environment variable (`no`/`yes`) before the reader is opened. If omitted, libnfc's own default is left untouched. |

> `-S` currently probes **block 0** with **10 nonce acquisitions** (fixed); `-P`/`-T`
> do not affect the static-nonce test. `-I` is useful when autodetection of the reader
> probes (or disturbs) devices you don't want touched.

#### Examples

```sh
# Static-nonce diagnostic only (no cracking): is this card a clone/magic chip?
mfoc-hardnested -S

# Force-disable libnfc intrusive scan, then dump the card
mfoc-hardnested -I 0 -O mycard.mfd

# Force-enable intrusive scan while running the static-nonce check
mfoc-hardnested -I 1 -S
```

Full usage string in this fork:

```
Usage: mfoc-hardnested [-h] [-C] [-F] [-S] [-k key] [-f file] ... [-P probnum] [-T tolerance] [-O output]
  ...
  I [0|1] force tag reader intrusive scan (0 = disable, 1 = enable)
  S       static-nonce diagnostic: collect plain nonces, report if tag nonce is constant (clone/magic)
```

---

## 2. Release note (for a GitHub release / CHANGELOG)

### vX.Y — static-nonce diagnostic & intrusive-scan control

**New**

- **`-S` — static-nonce diagnostic mode.** Fingerprints a Mifare Classic tag by
  collecting its plaintext `nt` nonces and checking whether they are constant.
  A constant nonce flags a likely clone / "magic" chip (e.g. Fudan FM11RF08S
  family). The mode is **read-only** and **requires no key** — it does not run
  any key-recovery attack. Inspired by the static-nonce / backdoor research
  published by Philippe Teuwen (Quarkslab, 2024) and mirrors the kind of
  fingerprint reported by Proxmark Iceman's `hf mf info`.
- **`-I [0|1]` — force libnfc intrusive scan on/off.** Exposes libnfc's
  `LIBNFC_INTRUSIVE_SCAN` setting on the command line (`0` = `no`, `1` = `yes`),
  so intrusive probing can be forced on or off regardless of the build/config
  default.

**Behaviour**

- With `-S`, the tool selects the tag, confirms it is Mifare Classic, runs the
  static-nonce test (block 0, 10 probes) and exits without cracking keys.
- Without `-I`, libnfc's own intrusive-scan default is unchanged.

**Notes**

- A "static nonce" verdict is a **heuristic** indicator of a non-genuine or
  special chip, not a cryptographic proof. "Variable nonce" is the expected
  result for a genuine Mifare Classic.

---

## 3. `detect_static_nonce()` — routine documentation

### Signature

```c
int detect_static_nonce(mftag t, mfreader r, uint8_t block, unsigned probes);
```

| Parameter | Meaning |
|-----------|---------|
| `t`       | Selected tag context (`mftag`). |
| `r`       | Reader context (`mfreader`). |
| `block`   | Block number targeted by the AUTH-A command used to elicit a nonce (called with `0`). |
| `probes`  | Number of nonce acquisitions. Clamped to `[2, 64]`: a value `< 2` becomes `5`, a value `> 64` becomes `64`. (`-S` calls it with `10`.) |

### Return value

| Value | Meaning |
|-------|---------|
| ` 1`  | **Static nonce** — `nt` was identical across all acquisitions → likely clone / magic chip (e.g. FM11RF08S family). |
| ` 0`  | **Variable nonce** — `nt` varied (pseudo-random) → standard genuine chip. |
| `-1`  | **Inconclusive** — fewer than 2 nonces could be collected, or a device/property error occurred. |

### How it works

The first tag nonce (`nt`) of a Mifare Classic authentication is sent by the
card **in clear**, before the Crypto-1 stream starts, so it can be read without
knowing any key. The routine exploits this:

1. Build an `AUTH A` command for `block` (`MC_AUTH_A, block, 0x00, 0x00`) and
   append the ISO 14443-A CRC (`iso14443a_crc_append`).
2. Clamp `probes` to `[2, 64]`.
3. Repeat `probes` times:
   - **Re-select the tag cleanly** (`mf_configure` + `mf_anticollision`). A
     genuine chip only emits a *fresh* `nt` after a new anticollision, so each
     probe starts from a clean selection.
   - Switch the reader to raw mode — disable CRC handling (`NP_HANDLE_CRC` off)
     and easy framing (`NP_EASY_FRAMING` off) — so the 4-byte `nt` is read
     unmodified.
   - Transceive the AUTH command and capture the response; re-enable easy
     framing afterwards.
   - If fewer than 4 bytes come back, log a missed probe and continue.
   - Otherwise store `nt = bytes_to_num(Rx, 4)`.
4. Restore normal reader config (`NP_HANDLE_CRC` + `NP_HANDLE_PARITY` on).
5. If fewer than 2 nonces were collected → return `-1` (inconclusive).
6. Compare every collected nonce to the first one:
   - All identical → print the static-nonce warning (constant `nt`, "likely
     clone / magic, e.g. FM11RF08S family") and return `1`.
   - Any difference → print "variable nonce → standard chip" and return `0`.

### Limitations & notes

- **Heuristic, not proof.** A constant `nt` strongly suggests a clone/magic or
  special chip, but some readers/antennas or edge timing conditions can also
  perturb acquisition. Treat the verdict as a fingerprint, not a guarantee.
- **Fixed parameters under `-S`.** The CLI always calls the routine with
  `block = 0` and `probes = 10`; `-P`/`-T` have no effect on this test.
- **Up to 64 nonces** are stored (internal `nonces[64]` buffer); the clamp keeps
  `probes` within that bound.
- **Side effects on reader state.** The routine toggles `NP_HANDLE_CRC`,
  `NP_EASY_FRAMING` and `NP_HANDLE_PARITY`; it restores CRC/parity on exit, and
  under `-S` the program exits immediately afterwards.
