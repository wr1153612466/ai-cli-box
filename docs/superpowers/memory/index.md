---
type: module_card
title: ai-cli-box-memory-index
summary: Initial memory index for the AI CLI Box Windows desktop app concept.
tags:
  - product
  - desktop
last_verified_commit: 02d2ff1
status: draft
---

# Repository Memory

## Covered Domains

- Product concept for a Windows/macOS desktop AI CLI workbench.
- UI direction captured in the Figma concept file and refined through packaged-app testing.
- Runtime architecture boundaries for Tauri, React, xterm.js, Rust PTY, local persistence, history, dashboard, commands, and task queue.
- Durable product lessons from the 2026-04-25 to 2026-04-28 long buildout session.

## Main Docs

- [Product and UI concept](product-ui-concept.md)
- [Runtime architecture boundaries](runtime-architecture-boundaries.md)
- [Buildout product lessons](buildout-product-lessons.md)

## Major Gaps

- Current checkout may be behind `origin/main`; the latest app code evidence is tied to commit `02d2ff1`.
- Claude, Gemini, and OpenCode history support is still future work; Codex history is the first implemented history source.
- Signing, auto-update, and release automation remain roadmap items.
