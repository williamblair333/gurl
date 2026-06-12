# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| latest `main` | ✅ |

## Reporting a Vulnerability

Please **do not** open a public GitHub issue for security vulnerabilities.

Email **williamblair333@gmail.com** with:
- A description of the vulnerability
- Steps to reproduce
- Potential impact

You'll receive a response within 72 hours. If the issue is confirmed, a fix will be released as quickly as possible and you'll be credited in the changelog.

## Known Security Considerations

- **Never put plaintext secrets in `config.yaml`** — use `"${ENV_VAR}"` interpolation. GURL will error on startup if an env var is missing rather than silently using an empty value.
- **`relay run` executes commands from a file** — only point it at files you control. GURL uses `shlex.split` with `shell=False` to prevent injection, but the commands themselves are executed as-written.
- **Network backends (HTTP, MQTT, Sonoff) transmit relay commands in plaintext** unless your broker/device is configured for TLS. Use a local network or VPN.
