# ADR-0001: Use Tauri, React, xterm.js, Rust PTY, SQLite, and keyring

**Date**: 2026-04-25
**Status**: accepted
**Deciders**: project owner, Codex

## Context

AI CLI Box is a Windows-first desktop workbench for managing multiple local AI CLI terminal sessions. The app needs to feel lightweight, responsive under long terminal output, safe around credentials, and flexible enough to implement the approved Figma UI. The core risk is terminal performance: AI CLI sessions can emit large output streams, so terminal buffers must not be stored in React state.

## Decision

We use Tauri v2 for the desktop shell, React and TypeScript for the UI, Zustand for UI metadata state, xterm.js for terminal rendering, Rust with `portable-pty` for process and PTY management, SQLite for non-secret session/settings metadata, and keyring-based storage for secrets.

Terminal output streams directly from the Rust PTY event bridge into xterm.js. React and Zustand store only metadata such as sessions, tabs, settings, tool profiles, and status values.

## Alternatives Considered

### Alternative 1: Electron, React, and node-pty
- **Pros**: Mature terminal ecosystem, widely used by VS Code-style tools, fast development path.
- **Cons**: Larger bundle, higher memory footprint, broader Node.js runtime exposure.
- **Why not**: The project prioritizes lightweight packaging and a narrow system-access boundary.

### Alternative 2: .NET WinUI 3
- **Pros**: Strong Windows-native UI integration and good OS-level performance.
- **Cons**: Slower iteration from Figma to UI, less reusable web UI ecosystem, more friction around terminal rendering and future cross-platform work.
- **Why not**: The project needs rapid UI iteration and a terminal-first web-rendered interface.

### Alternative 3: Flutter Desktop
- **Pros**: Consistent cross-platform UI and strong custom rendering.
- **Cons**: Terminal emulator integration and CLI process workflows are not its strongest path.
- **Why not**: The product is terminal-centric, and xterm.js is a better fit for terminal rendering.

### Alternative 4: Wails with React or Vue
- **Pros**: Lightweight WebView desktop app with a Go backend.
- **Cons**: Rust has a stronger fit for this project's process, PTY, and safety boundary needs.
- **Why not**: Tauri better matches the desired Rust-based backend boundary.

## Consequences

### Positive

- Smaller and lighter desktop footprint than Electron-style packaging.
- Clear separation between UI metadata state and high-volume terminal output.
- Rust backend can own process lifecycle, PTY IO, shell detection, filesystem access, and future packaging tasks.
- SQLite provides a local-first metadata store without requiring a server.
- Keyring storage keeps API keys, OAuth credentials, and proxy passwords out of plaintext settings and SQLite.

### Negative

- PTY event bridging must be implemented carefully because the mature `node-pty` path is not used.
- Tauri plugin permissions and Windows WebView2 behavior add platform-specific testing requirements.
- xterm.js lifecycle must be kept outside normal React rendering to avoid leaks and duplicate terminal instances.

### Risks

- **Risk**: Long AI CLI output causes UI jank if routed through React state.
  **Mitigation**: Stream PTY output directly into xterm.js and store only session metadata in Zustand.
- **Risk**: Secret handling is accidentally mixed with general settings.
  **Mitigation**: Keep secret APIs behind a dedicated keyring wrapper and store only secret references in metadata.
- **Risk**: PowerShell 7 is not installed on some Windows machines.
  **Mitigation**: Detect PowerShell 7, Windows PowerShell, cmd, and custom shell paths before session launch.

