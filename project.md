# AirTOTP — Air-gapped 2FA device firmware

## Vision

Custom firmware for the Raspberry Pi Zero 2W turning a SeedSigner-like device into a fully air-gapped TOTP generator (RFC 6238). No network connectivity, no plaintext persistent secrets. The security model is inspired by SeedSigner: stateless by default, secrets live only in RAM during an active session.

---

## Security Model

- **SD #1 (firmware)** — mounted read-only after boot. Contains only the OS and application code. No secrets ever stored here.
- **SD #2 (data)** — fully LUKS-encrypted partition (AES-256-XTS, argon2id KDF). Contains the TOTP seed database. Only mounted after passphrase entry at boot.
- **During session** — seeds are decrypted and loaded into RAM (`tmpfs`). SD #2 is unmounted immediately after loading and can be physically removed.
- **On shutdown** — RAM is cleared, no secret survives on the device.
- **Network** — disabled at kernel level (no Wi-Fi/BT modules loaded).

### Threat model

| Threat | Coverage |
|---|---|
| Device stolen while powered on | RAM wipe on shutdown |
| Device stolen while powered off | SD #2 encrypted, SD #1 secret-free |
| Forensic extraction of SD #2 | LUKS2 + argon2id |
| Evil maid attack on SD #1 | dm-verity (optional, Phase 3) |
| Network exfiltration | No network, no driver loaded |

### Out of scope

- Physical attack during active session (shoulder surfing, cold boot attack)
- Firmware supply chain compromise
- Weak passphrase

---

## Target Hardware

| Component | Reference | Notes |
|---|---|---|
| SBC | Raspberry Pi Zero 2W | ARM Cortex-A53 quad-core |
| Display | 1.3" TFT ST7789 240×240 | SPI, already used in SeedSigner builds |
| Buttons | 3× momentary push buttons | Up / Down / Select |
| RTC | DS3231 | I2C, ±2ppm accuracy, CR2032 backup battery |
| Secondary SD reader | USB micro-SD adapter | For SD #2 |
| Enclosure | 3D printed | Fork SeedSigner enclosure or design custom |

---

## Software Architecture

```
/
├── boot/                  # FAT32 — kernel, DTB, config.txt
├── rootfs/                # ext4 read-only (dm-verity optional)
│   ├── /app/              # AirTOTP application
│   │   ├── main.py        # Entry point
│   │   ├── totp.py        # RFC 6238 implementation
│   │   ├── storage.py     # LUKS management + RAM loading
│   │   ├── ui/
│   │   │   ├── screen.py  # ST7789 driver
│   │   │   ├── input.py   # Button handler (GPIO)
│   │   │   └── views/     # Screens: home, list, code, settings, unlock
│   │   └── rtc.py         # DS3231 I2C driver
│   ├── /etc/              # Minimal system config
│   └── /lib/              # Python 3.x + dependencies
└── (SD #2, external USB)
    └── luks_container
        └── totp.db        # SQLite database
```

---

## Build System

Use **Buildroot** to produce a minimal Linux image.

### Buildroot constraints

- Target: derive from `raspberrypi0_2w_defconfig`
- Init system: **BusyBox init** (no systemd — too heavy)
- Python 3.11+ with only required packages
- Kernel: disable all network modules (`CONFIG_WLAN`, `CONFIG_BT`, etc.)
- Rootfs: mount read-only (`ro` in `/etc/fstab`)
- Tmpfs: `/tmp`, `/run`, `/var` in RAM

### Required Buildroot packages

```
BR2_PACKAGE_PYTHON3=y
BR2_PACKAGE_PYTHON_PYOTP=y          # or vendor into /app
BR2_PACKAGE_CRYPTSETUP=y            # LUKS
BR2_PACKAGE_SQLITE=y
BR2_PACKAGE_PYTHON_SMBUS2=y         # I2C for DS3231
BR2_PACKAGE_RPI_GPIO=y              # GPIO buttons
BR2_PACKAGE_PILLOW=y                # TFT screen rendering
```

> If a package is not available in Buildroot, vendor it under `/app/vendor/`.

---

## Application — Detailed Specifications

### Boot sequence

```
1. Kernel boot (target: < 5s)
2. Init script:
   a. Mount rootfs read-only
   b. Mount tmpfs on /tmp, /run
   c. Initialize DS3231 RTC → sync system clock
   d. Detect SD #2 (USB)
   e. Launch /app/main.py
3. main.py:
   a. Display unlock screen
   b. Passphrase entry (see UX section)
   c. Open LUKS container → mount /mnt/data
   d. Load totp.db into RAM (/tmp/totp.db)
   e. Unmount /mnt/data (SD #2 can be physically removed)
   f. Main UI loop
```

### Module `totp.py`

Implement RFC 6238 from scratch or as a minimal wrapper around `pyotp`.

```python
# Expected interface
class TOTPEntry:
    id: str           # UUID v4
    label: str        # e.g. "GitHub - john@example.com"
    issuer: str       # e.g. "GitHub"
    secret: str       # Base32, kept in memory only
    digits: int       # 6 or 8
    period: int       # 30s default
    algorithm: str    # SHA1 / SHA256 / SHA512

def generate_code(entry: TOTPEntry, timestamp: int) -> str: ...
def time_remaining(period: int, timestamp: int) -> int: ...
```

### Module `storage.py`

```python
# Responsibilities:
# - Detect LUKS device on USB
# - Open/close LUKS container (via cryptsetup subprocess)
# - Load totp.db into /tmp after unlock
# - Save (re-encrypt) on changes
# - Zero out /tmp/totp.db on demand or at shutdown

def unlock(passphrase: str) -> bool: ...
def load_entries() -> list[TOTPEntry]: ...
def save_entries(entries: list[TOTPEntry]) -> None: ...
def wipe_ram_copy() -> None: ...
```

Storage format: SQLite (simple, well-supported, easy to extend).

### Module `rtc.py`

```python
# DS3231 driver via I2C (bus 1, address 0x68)
# Read time at boot → set system clock
# Write time on manual adjustment

def read_time() -> datetime: ...
def write_time(dt: datetime) -> None: ...
def sync_system_clock() -> None: ...
```

### UI Layer

**Principle**: state machine. Each screen is a class with `render()` and `handle_input(button)`.

```python
class View:
    def render(self, screen: Screen) -> None: ...
    def handle_input(self, button: Button) -> View | None: ...
    # Returns the next view, or None to stay on current
```

**Views to implement:**

| View | Description |
|---|---|
| `UnlockView` | LUKS passphrase entry, character by character |
| `HomeView` | Account list, up/down navigation |
| `CodeView` | 6-digit code display + countdown progress bar |
| `AddEntryView` | Add account (manual entry; QR scan optional) |
| `SettingsView` | Manual time adjustment, wipe, about |
| `ConfirmView` | Generic yes/no confirmation dialog |
| `ErrorView` | Error display |

**ST7789 240×240 display conventions:**

- Primary font: monospace, large enough to read at arm's length (24px+ for the TOTP code)
- Minimal palette: black background, white text, single accent color (orange or green)
- Progress bar: bottom of screen, drains over 30s
- No complex animations (limited CPU)

**Button mapping:**

```
UP button       → navigate up / increment character
DOWN button     → navigate down / decrement character
SELECT button   → confirm / enter submenu
Long press SELECT (2s) → back / cancel
```

---

## Passphrase Entry

Typing a complex passphrase with 3 buttons is the main UX constraint.

### Recommended: numeric PIN + strong argon2id

- 8–12 digit PIN
- Entry: up/down to change digit, select to confirm each digit
- KDF: argon2id with high parameters (`t=4, m=131072, p=4`) to compensate for lower entropy
- Display: masked as `****`

### Alternative: QR code passphrase

- Screen shows "Scan your unlock QR code"
- Camera (OV2640 or similar) scans a QR containing the full passphrase
- Passphrase is never typed manually
- Requires storing the unlock QR separately (smartphone, safe, etc.)

Start with the PIN approach in v1 — it requires no extra hardware.

---

## TOTP Database Schema

```sql
CREATE TABLE entries (
    id          TEXT PRIMARY KEY,  -- UUID v4
    label       TEXT NOT NULL,
    issuer      TEXT,
    secret      TEXT NOT NULL,     -- Base32
    digits      INTEGER DEFAULT 6,
    period      INTEGER DEFAULT 30,
    algorithm   TEXT DEFAULT 'SHA1',
    created_at  INTEGER,           -- Unix timestamp
    sort_order  INTEGER DEFAULT 0
);
```

---

## Adding a TOTP Account

### Via QR code (requires optional camera add-on)
- Scan an `otpauth://totp/...` URI
- Parse label, secret, issuer, digits, period
- Confirm and save

### Via manual entry
- Enter label (on-screen alphanumeric keyboard)
- Enter Base32 secret
- Confirm

> Camera is optional in v1. Prioritize manual entry to keep the initial hardware simple.

---

## Development Phases

### Phase 1 — Prototype (Raspberry Pi OS Lite)
**Goal: validate the full flow before investing in Buildroot**

- [ ] Python app running on stock Pi OS Lite
- [ ] ST7789 display driver working
- [ ] 3× GPIO buttons handled
- [ ] TOTP module (generation + display)
- [ ] DS3231 RTC driver
- [ ] LUKS on USB drive (manual CLI unlock first, then integrated)
- [ ] Full UI (all views)
- [ ] Complete flow: boot → unlock → account list → code display → shutdown

**Deliverable:** modified Pi OS Lite image, fully functional on target hardware.

### Phase 2 — OS Hardening
**Goal: replace Pi OS with a minimal Buildroot image**

- [ ] Configure Buildroot for Pi Zero 2W
- [ ] Reproduce Phase 1 on the Buildroot image
- [ ] Disable all network modules at kernel level
- [ ] Mount rootfs read-only
- [ ] BusyBox init script
- [ ] Boot time < 10s

**Deliverable:** bootable Buildroot image, network-disabled, read-only rootfs.

### Phase 3 — Advanced Security (optional)
- [ ] dm-verity on rootfs partition (SD #1 integrity)
- [ ] RAM zeroing on shutdown (shutdown hook)
- [ ] Secure boot (if supported on Pi Zero 2W)
- [ ] Full attack surface audit

---

## Repository Structure

```
airtopt/
├── buildroot/
│   ├── configs/
│   │   └── airtopt_defconfig       # Buildroot config derived from rpi0_2w
│   ├── board/
│   │   ├── rootfs_overlay/         # Files copied into final rootfs
│   │   │   ├── app/                # Application code
│   │   │   └── etc/                # System config (inittab, fstab, etc.)
│   │   └── post_build.sh
│   └── external.desc               # Buildroot external tree declaration
├── app/
│   ├── main.py
│   ├── totp.py
│   ├── storage.py
│   ├── rtc.py
│   └── ui/
│       ├── screen.py
│       ├── input.py
│       └── views/
│           ├── unlock.py
│           ├── home.py
│           ├── code.py
│           ├── add_entry.py
│           └── settings.py
├── tools/
│   ├── provision_data_sd.sh        # Initialize SD #2 (LUKS setup)
│   └── add_totp_entry.py           # CLI tool to add entries from a PC
├── docs/
│   ├── hardware.md                 # Wiring diagram
│   ├── threat_model.md             # Detailed threat model
│   └── build.md                    # Buildroot build instructions
├── tests/
│   └── test_totp.py                # RFC 6238 unit tests
└── README.md
```

---

## SD #2 Provisioning Script

```bash
#!/bin/bash
# tools/provision_data_sd.sh
# Initialize a blank SD card as an encrypted LUKS data partition

DEVICE=$1  # e.g. /dev/sdb

if [ -z "$DEVICE" ]; then
  echo "Usage: $0 /dev/sdX"
  exit 1
fi

echo "==> Creating LUKS2 partition on $DEVICE"
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  --pbkdf-memory 131072 \
  --pbkdf-time 4 \
  "$DEVICE"

echo "==> Opening container"
cryptsetup open "$DEVICE" airtopt_data

echo "==> Formatting ext4"
mkfs.ext4 /dev/mapper/airtopt_data

echo "==> Mounting and initializing database"
mkdir -p /mnt/airtopt_data
mount /dev/mapper/airtopt_data /mnt/airtopt_data

python3 -c "
import sqlite3
conn = sqlite3.connect('/mnt/airtopt_data/totp.db')
conn.execute('''
CREATE TABLE entries (
    id TEXT PRIMARY KEY,
    label TEXT NOT NULL,
    issuer TEXT,
    secret TEXT NOT NULL,
    digits INTEGER DEFAULT 6,
    period INTEGER DEFAULT 30,
    algorithm TEXT DEFAULT \"SHA1\",
    created_at INTEGER,
    sort_order INTEGER DEFAULT 0
)
''')
conn.commit()
conn.close()
print('Database initialized.')
"

umount /mnt/airtopt_data
cryptsetup close airtopt_data

echo "==> SD #2 ready."
```

---

## Python Dependencies

```
pyotp>=2.9.0          # TOTP/HOTP RFC 6238/4226
smbus2>=0.4.0         # I2C for DS3231
RPi.GPIO>=0.7.0       # GPIO buttons
Pillow>=10.0.0        # TFT screen rendering
spidev>=3.6           # SPI for ST7789
```

---

## Tests

```python
# tests/test_totp.py — RFC 6238 compliance check
# Official test vectors: RFC 6238 Appendix B

TEST_VECTORS = [
    # (secret, timestamp, expected_code, algorithm)
    ("12345678901234567890", 59,          "94287082", "SHA1"),
    ("12345678901234567890", 1111111109,  "07081804", "SHA1"),
    ("12345678901234567890", 1234567890,  "89005924", "SHA1"),
    ("12345678901234567890", 2000000000,  "69279037", "SHA1"),
    ("12345678901234567890", 20000000000, "65353130", "SHA1"),
]

def test_rfc6238_vectors():
    for secret, ts, expected, algo in TEST_VECTORS:
        result = generate_code_at(secret, ts, digits=8, algorithm=algo)
        assert result == expected, f"FAIL ts={ts}: got {result}, expected {expected}"
```

---

## Notes for Claude Code

- **Never log secrets** — seeds and passphrases must never appear in log files or stdout
- **Zero out secret variables** after use — reassign and `del` in Python
- **No network dependencies** — no `import urllib`, `requests`, or any network library
- **Inactivity timeout** — after 5 minutes idle on the unlock screen, blank the display
- **LUKS error handling** — wrong PIN shows a clear message, no hint about the passphrase
- **No swap partition** — verify the system has no swap (secrets must never be written to disk)
- **RTC missing** — if DS3231 is not detected, prompt for manual time entry before proceeding

---

## References

- [RFC 6238 — TOTP](https://www.rfc-editor.org/rfc/rfc6238)
- [RFC 4226 — HOTP](https://www.rfc-editor.org/rfc/rfc4226)
- [SeedSigner GitHub](https://github.com/SeedSigner/seedsigner)
- [Buildroot manual](https://buildroot.org/downloads/manual/manual.html)
- [cryptsetup / LUKS2](https://gitlab.com/cryptsetup/cryptsetup)
- [otpauth URI format](https://github.com/google/google-authenticator/wiki/Key-Uri-Format)
