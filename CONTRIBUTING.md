# Contributing to GURL

Thanks for wanting to contribute. GURL runs on community backends — if you have hardware we don't support yet, a PR is the best way to add it.

## Ways to Contribute

- **Add a backend** — new protocol or hardware type
- **Fix a bug** — open an issue first so we can discuss approach
- **Improve docs** — fix typos, add examples, clarify config
- **Report hardware compatibility** — tell us what you tested it on

## Getting Started

```bash
git clone https://github.com/williamblair333/gurl.git
cd gurl
uv sync --extra dev --extra serial --extra hid
uv run pytest          # make sure everything passes before you start
```

## Adding a Backend

1. Create `src/relay_control/backends/yourbackend.py`
2. Implement all methods of `BaseRelayBackend` (see `core/base.py`)
3. Add your optional dependency to `pyproject.toml`
4. Register the backend in `src/relay_control/core/registry.py`
5. Add unit tests in `tests/unit/backends/test_yourbackend.py`
6. Document it in `README.md` (protocol table + installation section)

The serial backend (`serial_tty.py`) is the simplest reference implementation.

## Code Standards

- **TDD** — write the failing test first, then the implementation
- **No shell=True** — use `shlex.split` + `shell=False` for subprocess calls
- **No plaintext secrets** — env var interpolation only
- Run `uv run pytest` and `uv run ruff check src/` before submitting

## Pull Request Checklist

- [ ] Tests pass (`uv run pytest`)
- [ ] Lint passes (`uv run ruff check src/`)
- [ ] New backend has unit tests with mocked hardware
- [ ] README updated if user-facing behaviour changed
- [ ] CHANGELOG.md updated

## License

By contributing, you agree your code will be licensed under GPL-3.0.
