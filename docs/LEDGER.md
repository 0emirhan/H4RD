# H4RD — Ledger Module

Security audit module for **Ledger Nano S / Nano X / Nano S+** hardware wallets.

---

## Supported Devices

| Model | VID:PID | Detection |
|-------|---------|-----------|
| Nano S | `2c97:0001` / `2c97:1011` | Auto |
| Nano X | `2c97:0004` / `2c97:4011` / `2c97:0006` | Auto |
| Nano S+ | `2c97:0005` | Auto |

---

## Checks (36 total)

### Recon
- USB Descriptor Analysis

### Firmware
- Firmware Version & Target ID
- CVE Database Lookup

### Supply Chain
- Supply Chain — USB Identity
- Supply Chain — Manufacturer
- Supply Chain — Unofficial FW

### App Verification
- App Hash Verification
- Suspicious App Detection
- Installed App Inventory

### Integrity
- Genuine Attestation (ECDSA challenge-response)

### Security
- Debug Mode Detection
- Bootloader Status
- PIN Brute-force Resistance

### Critical Vulnerabilities
- MCU Downgrade (CVE-2018-12048)
- Memory Isolation Audit
- BIP32 Derivation Audit
- TX Signing Abuse (LSB-004)
- App Sideload Protection
- Secure Element Probe

### Side-Channel Analysis
- Timing Side-Channel Analysis
- Clone Detection (Timing)

### Fuzzing
- APDU Interface Fuzzing

### Glitch / Fault Injection
- Response Coherence Analysis
- Rapid-Fire Stress Test
- State Machine Consistency
- Timing Spike Detection
- Replay Attack Resistance

### App Audit
- Bitcoin App Security Audit
- Ethereum App Security Audit

### Cryptographic Verification
- SCP Channel Establishment
- Ephemeral Key Entropy
- Nonce Uniqueness (GET_CHALLENGE)
- RNG Quality Assessment
- ECDSA Nonce Reuse Detection

### MCU / Secure Element
- MCU/SE Version Consistency
- MCU/SE Timing Fingerprint
- SE Response Entropy
- MCU Proxy Detection
- Firmware Downgrade (MCU/SE)

---

## Usage

```bash
h4rd scan --device ledger           # Full Ledger audit
h4rd scan --device ledger --quick   # Skip fuzzing & side-channel
h4rd scan --device ledger -o report.json
```

---

## Communication

Uses APDU over HID with channel `0x0101` and 64-byte packets. The module handles framing, CRC, and multi-packet reassembly automatically.

---

## Key CVEs Checked

| CVE | Severity | Description |
|-----|----------|-------------|
| CVE-2018-12048 | Critical | MCU firmware downgrade allowing code execution |
| LSB-004 | High | TX signing abuse via malformed APDU sequences |

---

## Requirements

```
hidapi
cryptography
ecdsa
```
