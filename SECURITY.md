# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 4.x | Yes |
| < 4.0 | No |

## Reporting a Vulnerability

If you discover a security vulnerability in H4RD, please report it responsibly:

1. **Do not** open a public GitHub issue
2. Send a detailed report to the repository maintainer via GitHub private vulnerability reporting
3. Include steps to reproduce, affected versions, and potential impact
4. You will receive an acknowledgement within 48 hours

## Scope

H4RD is a security audit tool designed for authorized testing. The following are **in scope** for vulnerability reports:

- Code execution vulnerabilities in H4RD itself
- Dependency vulnerabilities that affect H4RD users
- Data leakage from audit results or history files
- Authentication bypass in device communication layers

The following are **out of scope**:

- Vulnerabilities in target devices (Ledger, Trezor, etc.) — report those to the respective vendors
- Issues requiring physical access to the machine running H4RD
- Social engineering attacks

## Responsible Use

H4RD must only be used on devices you own or have explicit authorization to test. Unauthorized use against third-party devices is illegal and violates this project's terms of use.
