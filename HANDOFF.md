# GURL Handoff

## Current State

Project is fully designed and documented, repo is clean on `main`. No implementation code exists yet beyond the original `relay_control.py`, `relay_set.py`, and `run_commands.py` scripts (which remain functional for CH340 serial boards).

The design spec and Plan 1 are complete and approved. Implementation has not started.

## What Was Completed This Session

- Full design brainstorm and spec for GURL (Grand Unified Relay Logic)
- 10-backend plugin architecture designed and documented
- `README.md` overhauled with GURL branding, full protocol table, HA integration guide, CLI reference, architecture diagram, roadmap, backward compat table
- Implementation Plan 1 written (foundation + SerialBackend + CLI, TDD, 8 tasks)
- Repo renamed to `gurl` on GitHub and locally
- 18 GitHub topics added, repo description updated
- Two security issues identified in original scripts (shell injection in `relay_set.py:50` and `run_commands.py:80`) — both fixed in Plan 1's implementation

## Next Session

**Start here:** Execute Plan 1 from `docs/superpowers/plans/2026-06-11-gurl-plan-1-foundation.md`.

Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans`.

Plan 1 produces a working `relay` CLI that replaces the original scripts with the new plugin architecture, starting with the SerialBackend (CH340 hardware already available for smoke testing).

After Plan 1: write and execute Plan 2 (HID + Modbus + HTTP + MQTT backends), then Plan 3 (Daemon + HA MQTT Discovery).

## Most Important Thing

**The original scripts still work** — don't remove them until Plan 1 is implemented and smoke-tested on real CH340 hardware. Keep `relay_control.py`, `relay_set.py`, `run_commands.py` until the new `relay` CLI is verified.

## Environment Notes

- Local repo: `/opt/proj/reviews/gurl/`
- Remote: `https://github.com/williamblair333/gurl.git`
- Branch: `main` (clean, up to date with origin)
- No `pyproject.toml` yet — Plan 1 Task 1 creates it
- CH340 relay board available for hardware smoke tests (`@pytest.mark.hardware`)
