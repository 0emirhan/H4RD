# H4RD — Android Module

Security audit module for **Android devices** via ADB. Covers root detection, malware scanning, network exposure, crypto configuration, and exploit validation.

---

## Requirements

- ADB installed and in PATH
- USB debugging enabled on the device
- Device authorized for ADB

---

## Checks (27 total)

### Recon
- ADB Connection & Device Info

### Firmware
- Android Version & CVE Status
- Security Patch Age

### Security
- Root Detection
- Magisk Detection
- SELinux Status
- Verified Boot Status
- System Partition Integrity
- Full Disk / File-Based Encryption
- Screen Lock Strength
- Keystore Security
- Accessibility Services
- Unknown Sources Policy
- Developer Options Status
- USB Debug Authorization Audit
- Mock Location Detection

### Network
- ADB TCP Exposure
- Listening Ports Audit
- Firewall Status
- VPN Configuration
- DNS Configuration

### Malware / Privacy
- Suspicious Package Detection
- Dangerous Permissions Audit

### Cryptographic
- User CA Certificates
- Network Security Config
- Cleartext Traffic Detection
- Certificate Pinning Bypass

### Integrity
- System App Signature Integrity

### Exploit
- Exploit — ADB Shell Escalation
- Exploit — Wallet App Data Extraction

---

## Dangerous Permissions Monitored

```
READ_CALL_LOG          WRITE_CALL_LOG         PROCESS_OUTGOING_CALLS
READ_CONTACTS          READ_SMS               RECEIVE_SMS
SEND_SMS               CAMERA                 RECORD_AUDIO
ACCESS_FINE_LOCATION   ACCESS_BACKGROUND_LOCATION
READ_EXTERNAL_STORAGE  WRITE_EXTERNAL_STORAGE MANAGE_EXTERNAL_STORAGE
INSTALL_PACKAGES       REQUEST_INSTALL_PACKAGES
BIND_DEVICE_ADMIN      CHANGE_NETWORK_STATE   READ_PHONE_STATE
SYSTEM_ALERT_WINDOW    WRITE_SECURE_SETTINGS
```

---

## Suspicious Packages Detected

Spyware, stalkerware, root managers, and Xposed/LSPosed frameworks are automatically flagged.

---

## Android Version CVE Coverage

| Version | Status |
|---------|--------|
| 4.x–5.x | Critical — hundreds of unpatched CVEs |
| 6.x–7.x | High — end of life |
| 8.x–9 | Medium — end of life |
| 10–15 | Checked against live CVE database |

---

## Usage

```bash
h4rd scan --device android           # Full Android audit
h4rd scan --device android --quick   # Skip exploit checks
h4rd scan --device android -o report.json
```
