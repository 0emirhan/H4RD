# Installation Guide

## Requirements

- Python 3.10+
- libusb (system-level)
- hidapi (system-level)

## Quick Install

```bash
git clone https://github.com/0emirhan/H4RD && cd H4RD
pip install -r requirements.txt && pip install -e .
```

## Optional Dependencies

```bash
pip install cryptography ecdsa       # Full crypto verification (attestation, ECDSA nonce analysis)
pip install reportlab                 # PDF report generation
pip install pysha3                    # Faster Keccak-256 for Ethereum address derivation
```

## System Dependencies

### Linux (Debian/Ubuntu)

```bash
sudo apt install libusb-1.0-0-dev libhidapi-dev adb
```

**udev rules (required for non-root USB access):**

```bash
# Ledger
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2c97", MODE="0666"' | sudo tee /etc/udev/rules.d/99-ledger.rules

# Trezor
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="534c", MODE="0666"
SUBSYSTEM=="usb", ATTR{idVendor}=="1209", MODE="0666"' | sudo tee /etc/udev/rules.d/99-trezor.rules

sudo udevadm control --reload-rules && sudo udevadm trigger
```

### macOS

```bash
brew install libusb hidapi android-platform-tools
```

### Arch Linux

```bash
sudo pacman -S libusb hidapi android-tools
```

## Device-Specific Setup

### Ledger

- Install Ledger Live (optional, for firmware updates)
- Device must be unlocked and on the dashboard (not in an app)
- USB cable must support data transfer (not charge-only)

### Trezor

- Install Trezor Bridge or use direct HID access via udev rules
- Device must be on the home screen
- PIN entry required for most checks

### Android

- Enable Developer Options: Settings > About > tap Build Number 7 times
- Enable USB Debugging in Developer Options
- Authorize the computer when prompted on device
- Verify with: `adb devices`

### ChipWhisperer (optional, for glitch modules)

```bash
pip install chipwhisperer
```

Supported hardware: ChipWhisperer Husky, Pro, Lite

### BLE/NFC (optional, for wireless module)

```bash
sudo apt install bluez bluez-tools
```

Requires a Bluetooth 4.0+ adapter for BLE scanning.

## Verify Installation

```bash
h4rd list       # Should list connected devices
h4rd modules    # Should show all available audit modules
```
