---
name: Feature request / new backend
about: Suggest a new protocol, backend, or feature
labels: enhancement
---

**What hardware or protocol would this support?**
Be as specific as possible — model numbers, USB VID/PID, protocol spec links.

**Is there an existing Python library for this hardware?**
If yes, link it. This usually becomes the optional dependency.

**How would you use it?**
```bash
relay set mydevice 1 on
```
or
```yaml
devices:
  - id: mydevice
    backend: newbackend
    ...
```

**Are you willing to implement it?**
If yes, see [CONTRIBUTING.md](../../CONTRIBUTING.md) — the serial backend is a good reference.
