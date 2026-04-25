---
type: module_card
title: product-ui-concept
summary: Stable product and UI direction for the first AI CLI Box Windows desktop release.
tags:
  - product
  - ui
  - windows
owned_paths:
  - docs/superpowers/specs/
related_docs:
  - docs/superpowers/specs/2026-04-25-ai-cli-box-windows-desktop-ui-design.md
status: draft
---

# Product and UI Concept

## Responsibilities

- Define AI CLI Box as a Windows desktop workbench for managing multiple AI CLI terminal sessions.
- Keep the main workspace focused on project/session management and terminal work.
- Put AI tool configuration under Settings and tool selection inside New Session.
- Default to Simplified Chinese while supporting English through an i18n layer.

## Entry Points

- Figma concept: https://www.figma.com/design/SHxadGXK06XF11kg1EpN2H
- Design spec: `docs/superpowers/specs/2026-04-25-ai-cli-box-windows-desktop-ui-design.md`

## Invariants

- The UI is original and must not copy AICoder branding, icons, or exact visual styling.
- The left sidebar is for session search, new session, project groups, and session rows only.
- AI CLI tools are selected when creating a session and configured in Settings.
- The first Windows release targets Codex, Claude, Gemini, and OpenCode.
- Supported terminal shells include PowerShell 7, Windows PowerShell, cmd, and custom shell paths.
- The chosen foundation stack is Tauri v2, React, Zustand, xterm.js, Rust PTY, SQLite, and keyring-based secret storage.
- Terminal output is streamed into xterm.js and must not be stored in React or Zustand state.

## Extension Points

- Add more AI CLI tools through the same tool profile model.
- Add task queue, dashboard, and sync features after the core terminal workflow is stable.
- Add additional languages through the same i18n resource structure.

## Common Pitfalls

- Do not turn the main sidebar into a settings/navigation drawer.
- Do not make AI tool badges persistent in the main sidebar.
- Do not hardcode Chinese UI text in components; use localized resources.
- Do not store API keys or OAuth credentials in plaintext.
- Do not use React rendering for terminal output; it will degrade performance for long-running CLI sessions.
