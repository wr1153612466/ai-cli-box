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
  - README.md
last_verified_commit: 02d2ff1
status: draft
---

# Product and UI Concept

## Responsibilities

- Define AI CLI Box as a lightweight Windows/macOS desktop workbench for managing multiple AI CLI terminal sessions.
- Keep the main workspace focused on project/session management and terminal work.
- Put AI tool configuration under Settings and tool selection inside New Session.
- Default to Simplified Chinese while supporting English through an i18n layer.
- Preserve the native CLI feel: the app organizes terminals and workflows, but the running AI CLI remains the user's familiar terminal tool.

## Entry Points

- Figma concept: https://www.figma.com/design/SHxadGXK06XF11kg1EpN2H
- Design spec: `docs/superpowers/specs/2026-04-25-ai-cli-box-windows-desktop-ui-design.md`
- Product README: `README.md`

## Invariants

- The UI is original and must not copy AICoder branding, icons, or exact visual styling.
- The left sidebar is for session search, new session, project groups, and session rows only.
- AI CLI tools are selected when creating a session and configured in Settings.
- The first release targets Codex, Claude Code, Gemini CLI, and OpenCode.
- Supported terminal shells include PowerShell 7, Windows PowerShell, cmd, macOS/Linux zsh/bash/fish, and custom shell paths.
- The chosen foundation stack is Tauri v2, React, Zustand, xterm.js, Rust PTY, SQLite, and keyring-based secret storage.
- Terminal output is streamed into xterm.js and must not be stored in React or Zustand state.
- The application must not auto-create or auto-start terminal sessions on launch; users explicitly create or open sessions.
- Bottom/global entries are History, Commands, Task Queue, and Dashboard. Settings belongs in a separate settings surface, not as persistent sidebar navigation.
- Dashboard must not estimate dollar cost from model prices; price changes make that misleading. Show token/session/activity statistics instead.
- The terminal area has priority over surrounding UI. Sidebars, panels, and typography should stay compact so terminal output remains the main workspace.

## Extension Points

- Add more AI CLI tools through the same tool profile model.
- Add task queue, dashboard, and sync features after the core terminal workflow is stable.
- Add additional languages through the same i18n resource structure.
- Add fuller history support for Claude, Gemini, and OpenCode after Codex history behavior is reliable.

## Common Pitfalls

- Do not turn the main sidebar into a settings/navigation drawer.
- Do not make AI tool badges persistent in the main sidebar.
- Do not hardcode Chinese UI text in components; use localized resources.
- Do not store API keys or OAuth credentials in plaintext.
- Do not use React rendering for terminal output; it will degrade performance for long-running CLI sessions.
- Do not duplicate shell input fields outside the terminal; command/snippet insertion should write into the active terminal input exactly once.
- Do not add fake context, time, or usage indicators when the underlying CLI already presents its own state.
- Do not show duplicate custom window chrome if the native frame is visible.
