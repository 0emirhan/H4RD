# Contributing to H4RD

## Getting Started

1. Fork the repository
2. Clone your fork: `git clone https://github.com/<your-user>/H4RD && cd H4RD`
3. Install dev dependencies: `pip install -r requirements.txt && pip install -e .`
4. Create a branch: `git checkout -b feature/my-feature`

## Code Style

- Python 3.10+
- Use type hints for function signatures
- Follow existing module patterns (see `modules/wallets/ledger.py` as reference)
- Each audit check returns a dict with `status`, `details`, and optional `remediation`

## Adding a New Audit Check

1. Add the check method to the relevant module in `modules/`
2. Register it in `get_checks()` with a name, category, and function reference
3. Return a result dict following the existing format
4. Add the check to the corresponding device documentation in `docs/`

## Adding a New Exploit Module

1. Create a new file in `core/exploit/`
2. Use `ExploitLogger` mixin from `core/exploit/_logging.py`
3. Use `RobustHTTP` from `core/exploit/_http.py` for any API calls
4. Include `--dry-run` support for any destructive operations
5. Add CLI integration in `cli/main.py`
6. Document the module in `docs/EXPLOITS.md`

## Pull Requests

- One feature per PR
- Include a clear description of what and why
- Test on at least one device type (or in dry-run mode)
- Update documentation if adding new commands or checks

## Reporting Bugs

Open an issue with:
- Device type and firmware version
- H4RD version (`h4rd --version`)
- Full error output
- Steps to reproduce

## Security Vulnerabilities

Do **not** open a public issue. See [SECURITY.md](SECURITY.md) for responsible disclosure.
