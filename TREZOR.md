# H4RD — Trezor Module

Security audit module for **Trezor One / Model T** hardware wallets.

---

## Supported Devices

| Model | VID:PID | Detection |
|-------|---------|-----------|
| Trezor One | `534c:0001` | Auto |
| Trezor Model T | `1209:53c1` | Auto |

---

## Checks (38 total)

### Recon
- USB Descriptor Analysis

### Firmware
- Device Features Detection
- Firmware Version & CVE Check
- Bootloader Version & CVE Check

### Supply Chain
- Supply Chain — USB Identity
- Supply Chain — Clone Detection
- Supply Chain — Unofficial Firmware
- Supply Chain Full Audit

### Security
- Debug Link Detection
- PIN Protection Status
- Passphrase Protection (25th Word)
- Backup Status
- Wipe Code Protection
- State After Reset
- Session Leakage

### Attestation
- Firmware Signature Verification
- Bootloader Signature Verification
- Vendor Keys Integrity

### Side-Channel
- Side-Channel — PIN Variance
- Side-Channel — Differential
- Side-Channel — Bimodal Detection

### Fuzzing
- Protobuf Message Fuzzing
- GetPublicKey Edge Cases
- SignTx Malformed Inputs
- MessageSignature Abuse
- Unknown Message ID Sweep

### App Audit
- App Audit — Bitcoin Signing
- App Audit — Ethereum Signing
- App Audit — Multi-coin Path

### Exploit Modules
- Exploit — ECDSA Nonce Reuse
- Exploit — BIP32 Key Recovery
- Exploit — Debug Link Oracle
- Exploit — Protobuf Injection
- Exploit — Bellcore Fault Detection

### Glitch / Fault Injection
- Response Coherence Analysis
- Rapid-Fire Stress Test
- State Machine Consistency
- Timing Spike Detection
- Replay Attack Resistance

### MCU Analysis
- MCU Version Consistency
- MCU Timing Fingerprint
- MCU Response Entropy
- MCU Proxy / MitM Detection

---

## Usage

```bash
h4rd scan --device trezor           # Full Trezor audit
h4rd scan --device trezor --quick   # Skip fuzzing & side-channel
h4rd scan --device trezor -o report.json
```

---

## Communication

Uses Trezor wire protocol over USB HID (64-byte packets). Message framing follows the `?##` magic prefix with protobuf-encoded payloads.

| Message | ID | Description |
|---------|----|-------------|
| Initialize | 0 | Start session |
| Features | 17 | Device info response |
| Ping | 1 | Connectivity check |
| GetFeatures | 55 | Request device state |

---

## Requirements

```
hidapi
cryptography
ecdsa
```
