# H4RD — Generic USB Module

Security audit module for **any USB device**. Covers descriptor analysis, protocol fuzzing, BadUSB detection, and power profiling.

---

## Checks (14 total)

### Descriptor Analysis
- USB Class/Subclass Analysis
- String Descriptors Audit
- Configuration Count
- Interface & Endpoint Map

### Protocol Fuzzing
- Control Request Fuzzing (vendor requests `0x40`–`0x7F`)
- Malformed Descriptor Request
- Interface Claim Test
- Alternate Setting Sweep

### BadUSB Detection
- BadUSB — Dual Class Detection
- BadUSB — Descriptor Mismatch
- HID Report Descriptor Analysis

**Flagged combinations:**

| Classes | Risk |
|---------|------|
| HID + MassStorage | Potential BadUSB |
| CDC + HID | Possible keystroke injection |
| MassStorage + CDC | Potential rubber ducky |

### Power & Physical
- USB Power Draw Analysis
- USB Speed Negotiation

### Timing
- USB Response Timing Analysis

---

## Usage

```bash
h4rd scan --device usb              # Full USB audit
h4rd scan --device usb --quick      # Skip fuzzing & timing checks
h4rd scan --device usb -o report.json
```

---

## Quick Mode

When `--quick` is used, fuzzing and timing categories are skipped (8 checks instead of 14).

---

## Requirements

```
pyusb
libusb (system)
```
