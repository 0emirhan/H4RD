# Changelog

## v4.0.0 — Offensive Arsenal

### Audit Modules (126 total checks)

**Ledger — 39 checks**
- Fixed multi-packet HID response unwrapping (channel_id + tag + seq framing)
- Fixed PIN retry counter parsing (SW 0x63Cx extraction)
- Integrated `app_checker` for real hash verification against catalog
- Added real ECDSA attestation with device public key
- 12 categories: recon, supply chain, attestation, apps, security, seed/critical, side-channel, APDU fuzzing, glitch detection, app-specific, secure channel, MCU/SE consistency

**Trezor — 43 checks** (was 34)
- Added `send_raw_message()` for direct protobuf wire protocol
- Real firmware + bootloader signature verification (4-of-5 multisig)
- Real vendor key validation (5 secp256k1 public keys from SatoshiLabs)
- 5 glitch detection checks, 4 MCU consistency checks
- Welch's t-test for side-channel differential analysis
- Integrated `core.cve_db` (removed duplicate BOOTLOADER_CVE_DB)

**Android — 30 checks**
- Real per-app dangerous permission audit (top 10 via `dumpsys package`)
- App signature verification with cert hash extraction + AOSP test-key detection
- Network Security Config audit (SDK level, Private DNS, targetSdk)
- Cleartext traffic detection via `/proc/net/tcp` + 6 protocol checks
- USB Debug Authorization Audit (replaced duplicate check)
- Dynamic CVE lookup via `core.cve_db`
- DNS whitelist validation (Google/Cloudflare/Quad9/OpenDNS + RFC1918 + DoT/DoH)
- Firewall audit (INPUT/OUTPUT/FORWARD + ip6tables)

**USB Generic — 14 checks** (was 12)
- BOS descriptor awareness in malformed descriptor testing
- Per-class security evaluation in interface claim testing
- Crash detection via GET_STATUS probe in control fuzzing
- HID report descriptor parsing with usage page analysis
- USB timing analysis with CV-based implant detection
- Expanded string spoofing whitelist to 14 vendors + non-ASCII detection

### Offensive Exploit Modules (11 new)

| Module | Description |
|--------|-------------|
| `chain_orchestrator` | 8-phase automated exploit pipeline |
| `live_extraction` | BIP32/44 wallet tree + blockchain balance lookup |
| `post_exploit` | CVSS impact report + PoC TX crafting |
| `firmware_dump` | MCU extraction (BOLOS, SWD, DMA, UART) |
| `chipwhisperer_bridge` | CW glitch/CPA/SPA script generation |
| `wireless_audit` | BLE pairing, GATT, replay, NFC relay |
| `usb_attack` | HID injection, DuckyScript, USB gadget |
| `usb_proxy` | USB MitM APDU proxy + rule engine + PCAP |
| `blockchain_osint` | xpub OSINT, clustering, exchange tagging |
| `poc_generator` | 8 standalone PoC exploit scripts |
| `key_audit` | Defensive key blast radius assessment |

### Hardened Exploit Modules (12 critical fixes)

- **Keccak-256**: Created `_keccak.py` with correct 0x01 padding (not NIST SHA3 0x06). Fixed Pi step indexing, Chi step operator precedence, Theta step. All Ethereum address derivation now correct.
- **Error handling**: Created `_logging.py` ExploitLogger mixin, replaced bare `except: pass` with specific exception types across all 11 exploit modules.
- **HTTP resilience**: Created `_http.py` RobustHTTP with retry, backoff, rate limiting, 429 handling. Replaced raw urllib in 3 modules.
- **ECDSA signatures**: Added DER decode/encode, auto-format detection, device-specific parsers (Ledger DER, Trezor protobuf), low-s normalization (BIP 62), malleability detection.
- **Bellcore solver**: Full algebraic fault recovery + pure Python LLL lattice reduction + bit-flip enumeration with EC point verification.
- **DPA traces**: perf_counter_ns precision, Welch's t-test, template attack, cache-timing simulation, FFT filtering, KS test, adaptive sampling.
- **USB glitch**: 11 real phases including SOF alignment, EM fault simulation, bus contention, CW parameter output, session comparison.
- **Firmware dump**: Real BOLOS bootloader READ, SWD DPIDR probe, DMA leak (4 phases), voltage glitch prep.
- **USB proxy**: Full Bitcoin TX parsing, proper RLP for ETH (legacy + EIP-1559 + EIP-2930), rule chaining, PCAP-NG export, Rich live dashboard.
- **Wireless**: BLE pairing audit via bluetoothctl, GATT without pairing, APDU interception, replay testing, NFC relay resistance, spoofing detection.
- **PoC generator**: 8 complete standalone scripts with inline secp256k1, HID comm, --dry-run flag.
- **Blockchain OSINT**: Union-find clustering, common-input-ownership heuristic, exchange tagging, Chainalysis-compatible export.

### CLI (22 commands)

`scan`, `list`, `modules`, `info`, `cve-update`, `supply-chain`, `history`, `chain`, `extract`, `key-audit`, `post-exploit`, `craft-tx`, `firmware-dump`, `firmware-analyze`, `glitch-gen`, `glitch-run`, `wireless`, `usb-attack` (sub-app), `proxy`, `osint`, `poc`, `dpa`

- PDF export (`--pdf report.pdf`)
- `--no-history` flag
- `history` command for scan diff

### Infrastructure

- `core/attestation.py` — Real Root CA keys integrated
- `core/app_checker.py` — Dynamic hash loading from `~/.h4rd/app_hashes.json`
- `core/supply_chain.py` — Dynamic hash loading from `~/.h4rd/firmware_hashes.json`
- `reporters/pdf_reporter.py` — PDF report generation (requires reportlab)
