---
name: add-to-meal
description: Alias for /add-to-menu. Part of the yes-chef suite.
---

# /add-to-meal

Alias for `/add-to-menu`. Same behavior, same flow, same outputs.

**REQUIRED SKILL:** Invoke `/add-to-menu` via the Skill tool — its `SKILL.md` contains the complete protocol (target-file locating, delta grilling, appending sections, self-review). That skill in turn defers to `/plan-meal` for the underlying interview protocol.

This file exists purely to provide parity with the `/plan-meal` ↔ `/plan-menu` naming pair: either `-meal` or `-menu` suffix finds the right skill.

If `/add-to-menu` is unavailable for some reason, read `../add-to-menu/SKILL.md` directly and follow it verbatim.
