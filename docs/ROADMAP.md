# H4RD — Roadmap

Future modules and capabilities planned for development.

> All features are for authorized security research and personal device auditing only.

---

## Locked Device Extraction

### Ledger (Locked State)

| Module | Vecteur | Description |
|--------|---------|-------------|
| `locked_recon` | APDU | Enumerate all APDU commands that respond without PIN. Detect info leaks: SE version, internal state, app count, memory layout hints |
| `bootloader_exploit` | USB | Force bootloader entry (button combo + plug) and test unauthenticated memory read commands |
| `pin_bruteforce_timing` | Side-channel | Measure micro-timing variations on VERIFY PIN to detect per-digit correctness leaks |
| `usb_reset_glitch` | USB | Send USB resets at precise moments during boot to skip PIN verification (software-only, no ChipWhisperer) |
| `se_leak` | APDU | Send malformed APDUs to Secure Element to provoke error responses that leak internal state (oracle attack) |
| `dma_read` | APDU | Attempt out-of-bounds memory reads via crafted APDUs targeting MCU (not SE) |

**Constraints:**
- Nano S/X/S+ allow 3 PIN attempts before full wipe
- Direct bruteforce is dead, but timing side-channels may reveal weaknesses
- Bootloader memory read depends on firmware version and bootloader state
- SE oracle attacks have been patched in recent firmware but older versions may be vulnerable

---

### Android (Locked State)

Full memory extraction and security bypass for locked Android devices. Attack surface varies massively by chipset and security implementation.

#### Tier 1 — Feasible (no specialized hardware)

| Module | Target | Description |
|--------|--------|-------------|
| `adb_locked` | Any Android | Check if ADB is still active despite lock screen. If USB debugging was enabled before lock, ADB shell may be accessible for full dump |
| `mtk_dump` | MediaTek devices | BootROM exploit via `mtkclient`. Full flash dump on locked devices. Covers hundreds of models: Xiaomi, Realme, Oppo, Samsung A-series, Alcatel, ZTE |
| `edl_dump` | Qualcomm devices | Emergency Download (EDL) mode via Sahara/Firehose protocol. Full flash extraction if signed programmer (`.mbn`) is available. Covers majority of Android market |
| `bootloader_check` | Any Android | Test if bootloader is unlockable via `fastboot oem unlock`. If yes, flash custom recovery and dump `/data` partition |
| `recovery_dump` | Any Android | Boot into recovery mode, attempt to mount `/data`, verify if encryption holds. Extract unencrypted partitions (boot, system, vendor) |
| `legacy_dump` | Android < 7 | Older devices without FBE: `dd` raw dump of `/data` if no full-disk encryption or weak FDE implementation |

#### Tier 2 — Difficult but documented

| Module | Target | Description |
|--------|--------|-------------|
| `knox_audit` | Samsung (Knox) | Check Knox trip status (0x0 = clean, 0x1 = tripped), TIMA version, RKP status, TEE implementation, Knox Vault presence. Identify applicable kernel CVEs (e.g., CVE-2021-25337, CVE-2021-25369) |
| `qualcomm_tee` | Qualcomm (TrustZone) | Probe QSEE (Qualcomm Secure Execution Environment). Detect loaded trustlets, test for known TEE escapes (CVE-2019-10574, CVE-2021-1961). Check if secure boot chain is intact |
| `exynos_jtag` | Samsung Exynos | JTAG/SWD probe on Exynos chipsets. Some models have accessible debug pads on PCB. Can bypass secure boot if debug fuses not blown |
| `cold_boot` | Any (with removable battery or quick reboot) | Cold boot attack: freeze RAM to preserve contents, reboot into custom environment, dump RAM. Effective against FDE where key is held in memory. Guide + timing assistance |
| `huawei_testpoint` | Huawei (Kirin) | Testpoint identification + `PotatoNV` for bootloader unlock on Kirin chipsets. FBE still protects `/data` but boot/system/vendor are extractable |

#### Tier 3 — Near impossible (state of the art hardening)

| Target | Security Stack | Why |
|--------|---------------|-----|
| Pixel 6+ (Tensor + Titan M2) | Secure boot chain + FBE + hardware-bound keys + no known EDL mode | Keys are bound to Titan M2 hardware, cannot be extracted even with full flash dump |
| Samsung S21+ (Knox Vault) | Dedicated isolated security chip + anti-tamper + e-fuse | Physical tamper detection, separate processor for key storage, fuse-based rollback prevention |
| Samsung S24+ (Knox Vault + MDF) | Knox Vault + Message Guard + Auto Blocker | Additional firmware validation layers, real-time kernel protection |

---

### Encryption Analysis

| Module | Description |
|--------|-------------|
| `fbe_analyzer` | Detect encryption type (FDE vs FBE), identify CE (Credential Encrypted) vs DE (Device Encrypted) storage, test if DE keys are accessible without lock screen |
| `keymaster_probe` | Query Keymaster/StrongBox HAL version, test key attestation, identify if keys are hardware-bound or software-emulated |
| `dm_crypt_audit` | Analyze dm-crypt / dm-default-key configuration, check cipher strength, test for weak key derivation |
| `lockscreen_bypass` | Test known lock screen bypasses by Android version (emergency call exploits, notification tricks, accessibility service abuse) |

---

## MediaTek Deep Dive

Priority target — BootROM exploit is public and covers massive device range.

### mtkclient Integration

```
h4rd mtk-dump                        # Auto-detect MTK device, full flash dump
h4rd mtk-dump --partitions userdata  # Dump specific partition
h4rd mtk-dump --bypass-auth          # Use DA (Download Agent) authentication bypass
h4rd mtk-dump --preloader            # Dump preloader for further analysis
```

**Supported chipsets:**
- MT6735, MT6737, MT6739, MT6750, MT6755, MT6757
- MT6761, MT6762, MT6763, MT6765, MT6768, MT6769
- MT6771, MT6779, MT6781, MT6785
- MT6795, MT6797
- MT6833, MT6853, MT6873, MT6877, MT6885, MT6889, MT6893
- MT6983, MT6985

**Attack flow:**
1. Device enters BROM mode (hold Vol+ during plug)
2. `mtkclient` exploits BootROM to load custom Download Agent
3. DA has full flash access, bypasses secure boot
4. Dump entire EMMC/UFS including `userdata`
5. If FBE active, dump is encrypted — pass to `fbe_analyzer`

---

## Qualcomm EDL Deep Dive

### Firehose Integration

```
h4rd edl-dump                        # Auto-detect Qualcomm device in EDL mode
h4rd edl-dump --programmer prog.mbn  # Use specific signed programmer
h4rd edl-dump --rawxml               # Raw XML Firehose commands
h4rd edl-dump --peek 0x100000 4096   # Peek memory at address
```

**Entry methods:**
- ADB: `adb reboot edl`
- Fastboot: `fastboot oem edl`
- Hardware: short test points on PCB (device-specific)
- Exploit: certain bootloader vulnerabilities force EDL entry

**Programmer sources:**
- OEM firmware update packages (often contain `.mbn`)
- Leaked programmer databases
- Extracted from device OTA updates

---

## Samsung Knox Deep Dive

### Knox Security Layer Analysis

```
h4rd knox-status                     # Full Knox status report
h4rd knox-warranty                   # Warranty bit / trip status
h4rd knox-tee                        # TEE implementation details
h4rd knox-cve                        # Applicable CVEs for detected Knox version
```

**Knox components to audit:**
| Component | Function | Audit |
|-----------|----------|-------|
| TIMA (TrustZone Integrity Measurement) | Runtime kernel protection | Version detection, bypass history |
| RKP (Real-time Kernel Protection) | Prevents kernel code modification | Check if active, known escapes |
| Knox Vault | Hardware-isolated key storage | Presence detection, attestation check |
| Warranty Bit (e-fuse) | Tamper evidence | Read status, irreversibility warning |
| Secure Boot | Chain of trust from BL1 to Android | Verify each stage signature |
| DEFEX | Defeat Exploit protection | Policy audit, bypass testing |

---

## Google Titan Deep Dive

### Titan M / M2 Security Audit

```
h4rd titan-probe                     # Detect Titan M version and firmware
h4rd titan-cve                       # Check applicable CVEs
h4rd titan-attestation               # Key attestation verification
```

**Known CVEs:**
| CVE | Severity | Description |
|-----|----------|-------------|
| CVE-2022-20233 | Critical | Privilege escalation in Titan M firmware |
| CVE-2022-20244 | High | Information disclosure via Titan M |

**Audit scope:**
- Firmware version fingerprinting via Keymaster HAL
- Key attestation chain verification
- Bootloader lock state verification
- Verified boot state (green/yellow/orange/red)
- Android Verified Boot (AVB) hash validation

---

## Wireless Attack Surface (Locked Devices)

| Module | Description |
|--------|-------------|
| `ble_locked` | Scan for BLE services exposed while device is locked (notifications, fitness, wallet) |
| `nfc_locked` | Test NFC payment / tag emulation while locked (some devices allow NFC payments without unlock) |
| `wifi_probe` | Capture WiFi probe requests to fingerprint device (SSID history, MAC randomization quality) |
| `baseband_enum` | AT command injection via USB modem interface (some devices expose modem even when locked) |

---

## Infrastructure Needed

| Component | Purpose |
|-----------|---------|
| Device database | Chipset identification from model number, automated module selection |
| CVE tracker | Real-time CVE correlation for detected chipset + Android version + patch level |
| Programmer vault | Indexed collection of Qualcomm Firehose programmers by device model |
| MTK DA library | Download Agent binaries for MediaTek BootROM exploitation |
| Test lab inventory | Track which devices have been tested, results, firmware versions |

---

## Priority Order

1. **mtk_dump** — highest impact, public exploit, hundreds of device models
2. **edl_dump** — covers majority of Android market if programmers available
3. **adb_locked** — low effort, high reward when USB debug is on
4. **knox_audit** — Samsung is ~30% of Android market
5. **bootloader_check** — simple but effective recon
6. **fbe_analyzer** — needed to assess post-dump encryption
7. **locked_recon** (Ledger) — unique capability, no existing tool does this
8. **titan_probe** — Pixel market share growing, Titan M becoming standard
9. **cold_boot** — niche but devastating when applicable
10. **lockscreen_bypass** — version-dependent, catalog of known bypasses
