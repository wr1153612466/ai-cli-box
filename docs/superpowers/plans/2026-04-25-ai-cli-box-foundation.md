# AI CLI Box Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` to implement this plan task-by-task. It will decide whether each batch should run in parallel or serial subagent mode and will pass only task-local context to each subagent. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a runnable Windows-first Tauri + React foundation for AI CLI Box with the approved UI shell, i18n, session model, settings model, shell detection, and a testable terminal process boundary.

**Architecture:** The frontend owns layout, i18n, app metadata state, and user interactions. Terminal rendering is handled by `xterm.js` directly, while React/Zustand only stores session metadata and never stores terminal output buffers. The Tauri Rust backend owns OS-facing capabilities: shell detection, PTY lifecycle, process IO, filesystem paths, SQLite migrations, and secure secret boundaries.

**Tech Stack:** Tauri v2, React, TypeScript, Vite, Zustand, xterm.js, Vitest, React Testing Library, Rust, Cargo tests, `portable-pty`, Tauri SQL plugin with SQLite, keyring-based secret storage, plain scoped CSS.

---

## Scope Check

The approved spec covers several subsystems: desktop shell, UI, terminal sessions, settings, i18n, tool profiles, sync, notifications, and data management. This plan intentionally implements the first working foundation only:

- A runnable lightweight desktop app.
- The approved main UI shell.
- New Session flow model.
- Tool and shell selection.
- i18n resources for Simplified Chinese and English.
- Zustand app metadata store.
- xterm.js terminal renderer.
- Shell detection command.
- Rust PTY event bridge with tests and a minimal backend skeleton.
- SQLite schema for non-secret session/settings metadata.
- Keyring wrapper for API keys, OAuth credentials, and proxy passwords.

Sync, notification webhooks, full MCP editing UI, and packaging/update distribution should be separate follow-up plans after this foundation runs.

## File Structure

Create or modify these files:

- `package.json` - npm scripts and frontend dependencies.
- `vite.config.ts` - Vite and Vitest configuration.
- `src/main.tsx` - React entrypoint.
- `src/App.tsx` - application composition.
- `src/styles/tokens.css` - design tokens from the Figma concept.
- `src/styles/global.css` - base styles.
- `src/domain/tools.ts` - AI CLI tool registry and launch profile types.
- `src/domain/sessions.ts` - session, model, shell, and status types.
- `src/domain/settings.ts` - app, terminal, network, and sync settings types.
- `src/i18n/locales/zh-CN.ts` - Simplified Chinese strings.
- `src/i18n/locales/en-US.ts` - English strings.
- `src/i18n/index.tsx` - i18n provider and `useT` hook.
- `src/components/AppShell.tsx` - top chrome, sidebar, workspace, bottom status bar.
- `src/components/Sidebar.tsx` - session search, new session, project groups, session rows.
- `src/components/SessionTabs.tsx` - open session tabs.
- `src/components/TerminalPane.tsx` - terminal panel composition and command input boundary.
- `src/terminal/XtermTerminal.tsx` - xterm.js instance lifecycle; owns terminal buffer rendering.
- `src/terminal/useTerminalBridge.ts` - connects xterm input/output to Tauri events and commands.
- `src/components/StatusBar.tsx` - model and global actions.
- `src/components/NewSessionDialog.tsx` - first version of new session modal.
- `src/components/SettingsButton.tsx` - top-right utility controls.
- `src/state/appStore.ts` - Zustand store for session/settings metadata.
- `src/tauri/api.ts` - typed frontend wrappers for Tauri `invoke`.
- `src/db/schema.ts` - SQLite table and migration names used by the frontend/plugin boundary.
- `src/secrets/keyring.ts` - typed keyring wrapper for secret references.
- `src-tauri/Cargo.toml` - Rust dependencies.
- `src-tauri/src/lib.rs` - Tauri builder and command registration.
- `src-tauri/src/shells.rs` - Windows shell detection.
- `src-tauri/src/terminal.rs` - terminal session backend boundary.
- `src-tauri/src/session_store.rs` - SQLite migration registration for non-secret metadata.
- `src-tauri/src/models.rs` - serializable Rust DTOs.
- `tests/domain/session.test.ts` - frontend domain tests.
- `tests/i18n/i18n.test.tsx` - localization tests.
- `tests/components/sidebar.test.tsx` - sidebar behavior tests.
- `src-tauri/src/shells_tests.rs` - Rust shell detection tests.
- `src-tauri/src/terminal_tests.rs` - Rust terminal boundary tests.

---

### Task 1: Initialize Tauri React Project

**Files:**
- Create/Modify: generated Tauri + React project files
- Modify: `package.json`
- Modify: `vite.config.ts`

- [ ] **Step 1: Scaffold the project**

Run from `D:\project\ai-cli-box`:

```powershell
npm create tauri-app@2 . -- --template react-ts --manager npm
```

Expected: a Tauri v2 React TypeScript project is created in the current directory.

- [ ] **Step 2: Install test and UI dependencies**

```powershell
npm install
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
npm install @tauri-apps/api zustand @xterm/xterm @xterm/addon-fit @tauri-apps/plugin-sql tauri-plugin-keyring
```

Expected: `node_modules` is installed and `package-lock.json` is updated.

- [ ] **Step 3: Update `package.json` scripts**

Ensure `package.json` contains these scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "tauri": "tauri"
  }
}
```

- [ ] **Step 4: Configure Vitest in `vite.config.ts`**

Use this structure:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: "./src/test/setup.ts",
  },
});
```

- [ ] **Step 5: Add test setup**

Create `src/test/setup.ts`:

```ts
import "@testing-library/jest-dom/vitest";
```

- [ ] **Step 6: Verify the skeleton**

```powershell
npm run test
npm run build
```

Expected: tests pass or report no tests depending on scaffold defaults; build succeeds.

- [ ] **Step 7: Commit**

If `.git` exists:

```powershell
git add package.json package-lock.json vite.config.ts src src-tauri
git commit -m "chore: scaffold tauri react app"
```

If `.git` does not exist, skip the commit and record that the workspace is not a Git repository.

---

### Task 2: Define Domain Types And Defaults

**Files:**
- Create: `src/domain/tools.ts`
- Create: `src/domain/sessions.ts`
- Create: `src/domain/settings.ts`
- Test: `tests/domain/session.test.ts`

- [ ] **Step 1: Write frontend domain tests**

Create `tests/domain/session.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createSessionDraft, getToolById, TOOL_DEFINITIONS } from "../../src/domain/tools";
import { DEFAULT_TERMINAL_SETTINGS } from "../../src/domain/settings";

describe("AI CLI tool definitions", () => {
  it("includes the first release tools", () => {
    expect(TOOL_DEFINITIONS.map((tool) => tool.id)).toEqual([
      "codex",
      "claude",
      "gemini",
      "opencode",
    ]);
  });

  it("can create a Codex session draft with the default terminal shell", () => {
    const draft = createSessionDraft({
      toolId: "codex",
      projectPath: "D:\\project\\ai-cli-box",
      terminal: DEFAULT_TERMINAL_SETTINGS,
    });

    expect(draft.title).toBe("ai-cli-box");
    expect(draft.toolId).toBe("codex");
    expect(draft.shell.kind).toBe("powershell7");
    expect(draft.status).toBe("draft");
  });

  it("throws for unknown tools", () => {
    expect(() => getToolById("unknown")).toThrow("Unknown AI CLI tool: unknown");
  });
});
```

- [ ] **Step 2: Run test and verify failure**

```powershell
npm run test -- tests/domain/session.test.ts
```

Expected: FAIL because domain files do not exist.

- [ ] **Step 3: Create `src/domain/sessions.ts`**

```ts
export type AiToolId = "codex" | "claude" | "gemini" | "opencode";

export type ShellKind = "powershell7" | "windows-powershell" | "cmd" | "custom";

export interface ShellSelection {
  kind: ShellKind;
  label: string;
  executable: string;
  customPath?: string;
}

export type SessionStatus = "draft" | "running" | "stopped" | "failed";

export interface UsageSnapshot {
  modelLabel: string;
  contextPercent: number;
  timeLabel: string;
  weeklyLabel: string;
}

export interface AiSession {
  id: string;
  title: string;
  projectName: string;
  projectPath: string;
  toolId: AiToolId;
  shell: ShellSelection;
  modelLabel: string;
  launchProfileId: string;
  status: SessionStatus;
  updatedAt: string;
  usage: UsageSnapshot;
}

export interface ProjectGroup {
  id: string;
  name: string;
  path: string;
  sessions: AiSession[];
}
```

- [ ] **Step 4: Create `src/domain/settings.ts`**

```ts
import type { ShellSelection } from "./sessions";

export type AppLanguage = "zh-CN" | "en-US" | "system";
export type ThemeMode = "dark" | "light" | "system";

export interface TerminalSettings {
  defaultShell: ShellSelection;
  fontFamily: string;
  fontSize: number;
  theme: "graphite" | "one-dark" | "solarized" | "monokai";
  cursorStyle: "block" | "underline" | "bar";
  cursorBlink: boolean;
  lineHeight: number;
  letterSpacing: number;
  scrollbackLines: number;
}

export interface AppSettings {
  language: AppLanguage;
  theme: ThemeMode;
  closeBehavior: "ask" | "tray" | "exit";
  defaultProjectPath: string | null;
  sidebarSessionLimit: number;
}

export interface NetworkSettings {
  proxyHost: string;
  proxyPort: string;
  proxyUsername: string;
  proxyPasswordSet: boolean;
  noProxy: string;
}

export const DEFAULT_TERMINAL_SETTINGS: TerminalSettings = {
  defaultShell: {
    kind: "powershell7",
    label: "PowerShell 7",
    executable: "pwsh.exe",
  },
  fontFamily: "Fira Code",
  fontSize: 15,
  theme: "one-dark",
  cursorStyle: "bar",
  cursorBlink: true,
  lineHeight: 1.1,
  letterSpacing: 0,
  scrollbackLines: 10000,
};

export const DEFAULT_APP_SETTINGS: AppSettings = {
  language: "zh-CN",
  theme: "dark",
  closeBehavior: "ask",
  defaultProjectPath: null,
  sidebarSessionLimit: 5,
};

export const DEFAULT_NETWORK_SETTINGS: NetworkSettings = {
  proxyHost: "127.0.0.1",
  proxyPort: "7890",
  proxyUsername: "",
  proxyPasswordSet: false,
  noProxy: "localhost,127.0.0.1",
};
```

- [ ] **Step 5: Create `src/domain/tools.ts`**

```ts
import type { AiSession, AiToolId } from "./sessions";
import type { TerminalSettings } from "./settings";

export interface ToolDefinition {
  id: AiToolId;
  label: string;
  executable: string;
  accent: string;
  defaultModel: string;
}

export interface LaunchProfile {
  id: string;
  labelKey: string;
  args: string[];
}

export const TOOL_DEFINITIONS: ToolDefinition[] = [
  {
    id: "codex",
    label: "Codex",
    executable: "codex",
    accent: "#44D07B",
    defaultModel: "gpt-5.5 high",
  },
  {
    id: "claude",
    label: "Claude",
    executable: "claude",
    accent: "#FF7A45",
    defaultModel: "Auto",
  },
  {
    id: "gemini",
    label: "Gemini",
    executable: "gemini",
    accent: "#9C7DFF",
    defaultModel: "Auto",
  },
  {
    id: "opencode",
    label: "OpenCode",
    executable: "opencode",
    accent: "#9AA4B1",
    defaultModel: "Auto",
  },
];

export const LAUNCH_PROFILES: LaunchProfile[] = [
  { id: "balanced", labelKey: "launchProfile.balanced", args: [] },
  { id: "approval-required", labelKey: "launchProfile.approvalRequired", args: ["--ask-for-approval"] },
  { id: "sandbox-write", labelKey: "launchProfile.sandboxWrite", args: ["--sandbox", "workspace-write"] },
  { id: "autonomous", labelKey: "launchProfile.autonomous", args: ["--dangerously-bypass-approvals-and-sandbox"] },
];

export function getToolById(toolId: string): ToolDefinition {
  const tool = TOOL_DEFINITIONS.find((candidate) => candidate.id === toolId);
  if (!tool) {
    throw new Error(`Unknown AI CLI tool: ${toolId}`);
  }
  return tool;
}

export function projectNameFromPath(projectPath: string): string {
  const normalized = projectPath.replace(/\\+$/, "");
  const parts = normalized.split(/[\\/]/).filter(Boolean);
  return parts.at(-1) ?? "未命名项目";
}

export function createSessionDraft(input: {
  toolId: AiToolId;
  projectPath: string;
  terminal: TerminalSettings;
}): AiSession {
  const tool = getToolById(input.toolId);
  const projectName = projectNameFromPath(input.projectPath);
  return {
    id: crypto.randomUUID(),
    title: projectName,
    projectName,
    projectPath: input.projectPath,
    toolId: tool.id,
    shell: input.terminal.defaultShell,
    modelLabel: tool.defaultModel,
    launchProfileId: "balanced",
    status: "draft",
    updatedAt: new Date().toISOString(),
    usage: {
      modelLabel: tool.defaultModel,
      contextPercent: 100,
      timeLabel: "5h 91%",
      weeklyLabel: "本周 66%",
    },
  };
}
```

- [ ] **Step 6: Run tests**

```powershell
npm run test -- tests/domain/session.test.ts
```

Expected: PASS.

- [ ] **Step 7: Commit**

```powershell
git add src/domain tests/domain
git commit -m "feat: define app domain model"
```

Skip commit if `.git` is unavailable.

---

### Task 3: Add I18n Provider And Resources

**Files:**
- Create: `src/i18n/locales/zh-CN.ts`
- Create: `src/i18n/locales/en-US.ts`
- Create: `src/i18n/index.tsx`
- Test: `tests/i18n/i18n.test.tsx`

- [ ] **Step 1: Write i18n tests**

Create `tests/i18n/i18n.test.tsx`:

```tsx
import { renderHook } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import { I18nProvider, useT } from "../../src/i18n";

describe("i18n", () => {
  it("defaults to Simplified Chinese", () => {
    const { result } = renderHook(() => useT(), {
      wrapper: ({ children }) => <I18nProvider>{children}</I18nProvider>,
    });

    expect(result.current.t("app.workspace")).toBe("工作台");
    expect(result.current.language).toBe("zh-CN");
  });

  it("supports English", () => {
    const { result } = renderHook(() => useT(), {
      wrapper: ({ children }) => <I18nProvider initialLanguage="en-US">{children}</I18nProvider>,
    });

    expect(result.current.t("app.workspace")).toBe("Workspace");
  });

  it("falls back to the key when a translation is missing", () => {
    const { result } = renderHook(() => useT(), {
      wrapper: ({ children }) => <I18nProvider>{children}</I18nProvider>,
    });

    expect(result.current.t("missing.key")).toBe("missing.key");
  });
});
```

- [ ] **Step 2: Run test and verify failure**

```powershell
npm run test -- tests/i18n/i18n.test.tsx
```

Expected: FAIL because i18n files do not exist.

- [ ] **Step 3: Create `src/i18n/locales/zh-CN.ts`**

```ts
export const zhCN = {
  "app.workspace": "工作台",
  "app.name": "AI CLI Box",
  "sidebar.search": "搜索会话...",
  "sidebar.newSession": "新建会话",
  "sidebar.history": "历史",
  "sidebar.delete": "删除",
  "status.command": "指令",
  "status.taskQueue": "任务队列",
  "status.dashboard": "仪表盘",
  "status.context": "上下文 {value}%",
  "newSession.title": "新建 AI CLI 会话",
  "newSession.description": "选择工具、项目路径和启动模式。",
  "newSession.tool": "工具",
  "newSession.projectPath": "项目路径",
  "newSession.shell": "终端 Shell",
  "newSession.create": "创建",
  "newSession.cancel": "取消",
  "settings.title": "设置",
  "settings.tools": "AI 工具",
  "settings.terminal": "终端",
  "settings.general": "通用",
  "settings.network": "网络",
  "settings.dataSync": "数据与同步",
  "launchProfile.balanced": "平衡模式",
  "launchProfile.approvalRequired": "需要审批",
  "launchProfile.sandboxWrite": "沙箱可写",
  "launchProfile.autonomous": "自动模式",
} as const;
```

- [ ] **Step 4: Create `src/i18n/locales/en-US.ts`**

```ts
export const enUS = {
  "app.workspace": "Workspace",
  "app.name": "AI CLI Box",
  "sidebar.search": "Search sessions...",
  "sidebar.newSession": "New session",
  "sidebar.history": "History",
  "sidebar.delete": "Delete",
  "status.command": "Command",
  "status.taskQueue": "Task Queue",
  "status.dashboard": "Dashboard",
  "status.context": "Context {value}%",
  "newSession.title": "Create AI CLI Session",
  "newSession.description": "Choose a tool, project path, and launch mode.",
  "newSession.tool": "Tool",
  "newSession.projectPath": "Project path",
  "newSession.shell": "Terminal shell",
  "newSession.create": "Create",
  "newSession.cancel": "Cancel",
  "settings.title": "Settings",
  "settings.tools": "AI Tools",
  "settings.terminal": "Terminal",
  "settings.general": "General",
  "settings.network": "Network",
  "settings.dataSync": "Data & Sync",
  "launchProfile.balanced": "Balanced",
  "launchProfile.approvalRequired": "Approval required",
  "launchProfile.sandboxWrite": "Sandbox write",
  "launchProfile.autonomous": "Autonomous",
} as const;
```

- [ ] **Step 5: Create `src/i18n/index.tsx`**

```tsx
import { createContext, useContext, useMemo, useState, type ReactNode } from "react";
import { enUS } from "./locales/en-US";
import { zhCN } from "./locales/zh-CN";

export type Language = "zh-CN" | "en-US";
type TranslationKey = keyof typeof zhCN;
type Dictionary = Record<string, string>;

const dictionaries: Record<Language, Dictionary> = {
  "zh-CN": zhCN,
  "en-US": enUS,
};

interface I18nContextValue {
  language: Language;
  setLanguage: (language: Language) => void;
  t: (key: string, params?: Record<string, string | number>) => string;
}

const I18nContext = createContext<I18nContextValue | null>(null);

function format(template: string, params?: Record<string, string | number>): string {
  if (!params) return template;
  return Object.entries(params).reduce(
    (result, [key, value]) => result.replaceAll(`{${key}}`, String(value)),
    template,
  );
}

export function I18nProvider({
  children,
  initialLanguage = "zh-CN",
}: {
  children: ReactNode;
  initialLanguage?: Language;
}) {
  const [language, setLanguage] = useState<Language>(initialLanguage);

  const value = useMemo<I18nContextValue>(() => {
    const dictionary = dictionaries[language];
    return {
      language,
      setLanguage,
      t: (key, params) => format(dictionary[key as TranslationKey] ?? key, params),
    };
  }, [language]);

  return <I18nContext.Provider value={value}>{children}</I18nContext.Provider>;
}

export function useT(): I18nContextValue {
  const context = useContext(I18nContext);
  if (!context) {
    throw new Error("useT must be used inside I18nProvider");
  }
  return context;
}
```

- [ ] **Step 6: Run tests**

```powershell
npm run test -- tests/i18n/i18n.test.tsx
```

Expected: PASS.

- [ ] **Step 7: Commit**

```powershell
git add src/i18n tests/i18n
git commit -m "feat: add bilingual i18n foundation"
```

Skip commit if `.git` is unavailable.

---

### Task 4: Add Zustand App Metadata Store

**Files:**
- Create: `src/state/appStore.ts`
- Test: `tests/domain/app-store.test.ts`

- [ ] **Step 1: Write store tests**

Create `tests/domain/app-store.test.ts`:

```ts
import { beforeEach, describe, expect, it } from "vitest";
import { createInitialAppState, useAppStore } from "../../src/state/appStore";

describe("appStore", () => {
  beforeEach(() => {
    useAppStore.setState(createInitialAppState(), true);
  });

  it("starts with one sample Codex session for the approved UI shell", () => {
    const state = useAppStore.getState();
    expect(state.activeSessionId).toBe("session-codex-ai-cli-box");
    expect(state.projects[0].sessions[0].title).toBe("ai-cli-box");
  });

  it("opens and activates a session", () => {
    useAppStore.getState().openSession("session-claude-docs");
    const state = useAppStore.getState();
    expect(state.activeSessionId).toBe("session-claude-docs");
    expect(state.openSessionIds).toContain("session-claude-docs");
  });

  it("keeps terminal output outside app metadata state", () => {
    const state = useAppStore.getState();
    expect("terminalOutput" in state).toBe(false);
  });
});
```

- [ ] **Step 2: Run test and verify failure**

```powershell
npm run test -- tests/domain/app-store.test.ts
```

Expected: FAIL because `appStore.ts` does not exist.

- [ ] **Step 3: Create `src/state/appStore.ts`**

```ts
import { create } from "zustand";
import type { AiSession, ProjectGroup } from "../domain/sessions";
import { DEFAULT_APP_SETTINGS, DEFAULT_NETWORK_SETTINGS, DEFAULT_TERMINAL_SETTINGS } from "../domain/settings";

export interface AppState {
  projects: ProjectGroup[];
  openSessionIds: string[];
  activeSessionId: string;
  appSettings: typeof DEFAULT_APP_SETTINGS;
  terminalSettings: typeof DEFAULT_TERMINAL_SETTINGS;
  networkSettings: typeof DEFAULT_NETWORK_SETTINGS;
  isNewSessionOpen: boolean;
  openSession: (sessionId: string) => void;
  closeSession: (sessionId: string) => void;
  setNewSessionOpen: (open: boolean) => void;
}

function sampleSession(input: Partial<AiSession> & Pick<AiSession, "id" | "title" | "toolId" | "projectPath">): AiSession {
  return {
    projectName: "ai-cli-box",
    shell: DEFAULT_TERMINAL_SETTINGS.defaultShell,
    modelLabel: input.toolId === "codex" ? "gpt-5.5 high" : "Auto",
    launchProfileId: "balanced",
    status: "running",
    updatedAt: new Date().toISOString(),
    usage: {
      modelLabel: input.toolId === "codex" ? "gpt-5.5 high" : "Auto",
      contextPercent: 100,
      timeLabel: "5h 91%",
      weeklyLabel: "本周 66%",
    },
    ...input,
  };
}

export function createInitialAppState(): AppState {
  const sessions = [
    sampleSession({
      id: "session-codex-ai-cli-box",
      title: "ai-cli-box",
      toolId: "codex",
      projectPath: "D:\\project\\ai-cli-box",
    }),
    sampleSession({
      id: "session-claude-docs",
      title: "文档",
      toolId: "claude",
      projectPath: "D:\\project\\ai-cli-box",
    }),
  ];

  return {
    projects: [
      {
        id: "project-ai-cli-box",
        name: "ai-cli-box",
        path: "D:\\project\\ai-cli-box",
        sessions,
      },
    ],
    openSessionIds: ["session-codex-ai-cli-box", "session-claude-docs"],
    activeSessionId: "session-codex-ai-cli-box",
    appSettings: DEFAULT_APP_SETTINGS,
    terminalSettings: DEFAULT_TERMINAL_SETTINGS,
    networkSettings: DEFAULT_NETWORK_SETTINGS,
    isNewSessionOpen: false,
    openSession: () => undefined,
    closeSession: () => undefined,
    setNewSessionOpen: () => undefined,
  };
}

export const useAppStore = create<AppState>()((set, get) => ({
  ...createInitialAppState(),
  openSession: (sessionId) =>
    set((state) => ({
      activeSessionId: sessionId,
      openSessionIds: state.openSessionIds.includes(sessionId)
        ? state.openSessionIds
        : [...state.openSessionIds, sessionId],
    })),
  closeSession: (sessionId) =>
    set((state) => {
      const openSessionIds = state.openSessionIds.filter((id) => id !== sessionId);
      return {
        openSessionIds,
        activeSessionId: state.activeSessionId === sessionId ? openSessionIds[0] ?? "" : state.activeSessionId,
      };
    }),
  setNewSessionOpen: (open) => set({ isNewSessionOpen: open }),
}));
```

- [ ] **Step 4: Run tests**

```powershell
npm run test -- tests/domain/app-store.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/state tests/domain/app-store.test.ts
git commit -m "feat: add zustand app metadata store"
```

Skip commit if `.git` is unavailable.

---

### Task 5: Implement Approved App Shell UI

**Files:**
- Create: `src/styles/tokens.css`
- Create: `src/styles/global.css`
- Create: `src/components/AppShell.tsx`
- Create: `src/components/Sidebar.tsx`
- Create: `src/components/SessionTabs.tsx`
- Create: `src/components/TerminalPane.tsx`
- Create: `src/terminal/XtermTerminal.tsx`
- Create: `src/terminal/useTerminalBridge.ts`
- Create: `src/components/StatusBar.tsx`
- Create: `src/components/SettingsButton.tsx`
- Modify: `src/App.tsx`
- Modify: `src/main.tsx`
- Test: `tests/components/sidebar.test.tsx`

- [ ] **Step 1: Write sidebar behavior test**

Create `tests/components/sidebar.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { beforeEach, describe, expect, it } from "vitest";
import { Sidebar } from "../../src/components/Sidebar";
import { createInitialAppState, useAppStore } from "../../src/state/appStore";
import { I18nProvider } from "../../src/i18n";

describe("Sidebar", () => {
  beforeEach(() => {
    useAppStore.setState(createInitialAppState(), true);
  });

  it("shows session management without AI tool settings navigation", () => {
    render(
      <I18nProvider>
        <Sidebar />
      </I18nProvider>,
    );

    expect(screen.getByPlaceholderText("搜索会话...")).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "新建会话" })).toBeInTheDocument();
    expect(screen.getByText("ai-cli-box")).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Codex" })).not.toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Claude" })).not.toBeInTheDocument();
  });

  it("opens the new session dialog", async () => {
    const user = userEvent.setup();
    render(
      <I18nProvider>
        <Sidebar />
      </I18nProvider>,
    );

    await user.click(screen.getByRole("button", { name: "新建会话" }));

    expect(useAppStore.getState().isNewSessionOpen).toBe(true);
  });
});
```

- [ ] **Step 2: Run test and verify failure**

```powershell
npm run test -- tests/components/sidebar.test.tsx
```

Expected: FAIL because components do not exist.

- [ ] **Step 3: Add design tokens in `src/styles/tokens.css`**

```css
:root {
  --color-app: #101317;
  --color-top: #15191e;
  --color-sidebar: #12161b;
  --color-panel: #181d23;
  --color-panel-2: #1e242b;
  --color-input: #111820;
  --color-terminal: #0a0e13;
  --color-border: #2c343d;
  --color-border-strong: #394450;
  --color-text: #e7eaee;
  --color-muted: #8c96a3;
  --color-faint: #5f6874;
  --color-blue: #4d8dff;
  --color-green: #44d07b;
  --color-orange: #ff7a45;
  --color-purple: #9c7dff;
  --radius-sm: 7px;
  --radius-md: 10px;
  --radius-lg: 14px;
  --topbar-height: 42px;
  --sidebar-width: 252px;
  font-family: "Inter", "Microsoft YaHei", "Noto Sans SC", system-ui, sans-serif;
}
```

- [ ] **Step 4: Add base styles in `src/styles/global.css`**

```css
@import "./tokens.css";

* {
  box-sizing: border-box;
}

html,
body,
#root {
  width: 100%;
  height: 100%;
  margin: 0;
  overflow: hidden;
  background: var(--color-app);
  color: var(--color-text);
}

button,
input {
  font: inherit;
}

button {
  border: 0;
  cursor: pointer;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

- [ ] **Step 5: Create `src/components/Sidebar.tsx`**

```tsx
import { useT } from "../i18n";
import { useAppStore } from "../state/appStore";

export function Sidebar() {
  const { t } = useT();
  const projects = useAppStore((state) => state.projects);
  const activeSessionId = useAppStore((state) => state.activeSessionId);
  const openSession = useAppStore((state) => state.openSession);
  const setNewSessionOpen = useAppStore((state) => state.setNewSessionOpen);
  const project = projects[0];

  return (
    <aside className="sidebar" aria-label="会话">
      <div className="sidebar__search-row">
        <input className="sidebar__search" placeholder={t("sidebar.search")} />
        <button className="sidebar__icon-button" aria-label={t("sidebar.history")}>◷</button>
      </div>
      <div className="sidebar__new-row">
        <button
          className="sidebar__new-button"
          onClick={() => setNewSessionOpen(true)}
        >
          <span>＋</span>
          {t("sidebar.newSession")}
        </button>
        <button className="sidebar__icon-button" aria-label={t("sidebar.delete")}>⌫</button>
      </div>
      <div className="sidebar__project">
        <div className="sidebar__project-header">
          <span>⌄</span>
          <span>▣</span>
          <strong>{project.name}</strong>
          <span>{project.sessions.length}</span>
        </div>
        {project.sessions.map((session, index) => (
          <button
            className={session.id === activeSessionId ? "sidebar__session sidebar__session--active" : "sidebar__session"}
            key={session.id}
            onClick={() => openSession(session.id)}
          >
            <span className="sidebar__status-dot" />
            <span>{session.title}</span>
            <time>{index === 0 ? "1分钟前" : "23分钟前"}</time>
          </button>
        ))}
      </div>
    </aside>
  );
}
```

- [ ] **Step 6: Create `src/components/SessionTabs.tsx`**

```tsx
import { TOOL_DEFINITIONS } from "../domain/tools";
import type { AiSession } from "../domain/sessions";

export function SessionTabs({ sessions, activeSessionId }: { sessions: AiSession[]; activeSessionId: string }) {
  return (
    <nav className="tabs" aria-label="打开的会话">
      {sessions.map((session) => {
        const tool = TOOL_DEFINITIONS.find((candidate) => candidate.id === session.toolId);
        return (
          <button className={session.id === activeSessionId ? "tab tab--active" : "tab"} key={session.id}>
            <span className="tab__dot" style={{ backgroundColor: tool?.accent }} />
            <span>{tool?.label} / {session.title}</span>
            <span className="tab__close">×</span>
          </button>
        );
      })}
    </nav>
  );
}
```

- [ ] **Step 7: Create `src/terminal/useTerminalBridge.ts`**

```ts
import { listen, type UnlistenFn } from "@tauri-apps/api/event";
import { resizeTerminalSession, startTerminalSession, writeTerminalInput } from "../tauri/api";

export interface TerminalBridgeInput {
  sessionId: string;
  executable: string;
  cwd: string;
  args: string[];
  onData: (data: string) => void;
}

export async function attachTerminalBridge(input: TerminalBridgeInput): Promise<{
  dispose: () => Promise<void>;
  write: (data: string) => Promise<void>;
  resize: (cols: number, rows: number) => Promise<void>;
}> {
  const eventName = `terminal://data/${input.sessionId}`;
  const unlisten: UnlistenFn = await listen<string>(eventName, (event) => input.onData(event.payload));

  await startTerminalSession({
    sessionId: input.sessionId,
    executable: input.executable,
    cwd: input.cwd,
    args: input.args,
  });

  return {
    dispose: async () => {
      unlisten();
    },
    write: (data) => writeTerminalInput({ sessionId: input.sessionId, data }),
    resize: (cols, rows) => resizeTerminalSession({ sessionId: input.sessionId, cols, rows }),
  };
}
```

- [ ] **Step 8: Create `src/terminal/XtermTerminal.tsx`**

```tsx
import { FitAddon } from "@xterm/addon-fit";
import { Terminal } from "@xterm/xterm";
import "@xterm/xterm/css/xterm.css";
import { useEffect, useRef } from "react";
import type { AiSession } from "../domain/sessions";
import { attachTerminalBridge } from "./useTerminalBridge";

export function XtermTerminal({ session }: { session: AiSession }) {
  const containerRef = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const terminal = new Terminal({
      cursorBlink: true,
      cursorStyle: "bar",
      fontFamily: "Fira Code, Consolas, monospace",
      fontSize: 15,
      lineHeight: 1.1,
      scrollback: 10000,
      theme: {
        background: "#0A0E13",
        foreground: "#E7EAEE",
        cursor: "#44D07B",
      },
    });
    const fitAddon = new FitAddon();
    terminal.loadAddon(fitAddon);
    terminal.open(containerRef.current);
    fitAddon.fit();

    let disposed = false;
    let cleanup: (() => Promise<void>) | null = null;
    attachTerminalBridge({
      sessionId: session.id,
      executable: session.shell.executable,
      cwd: session.projectPath,
      args: [],
      onData: (data) => terminal.write(data),
    }).then((bridge) => {
      if (disposed) {
        void bridge.dispose();
        return;
      }
      cleanup = bridge.dispose;
      terminal.onData((data) => void bridge.write(data));
      terminal.onResize(({ cols, rows }) => void bridge.resize(cols, rows));
    });

    const resizeObserver = new ResizeObserver(() => fitAddon.fit());
    resizeObserver.observe(containerRef.current);

    return () => {
      disposed = true;
      resizeObserver.disconnect();
      terminal.dispose();
      if (cleanup) void cleanup();
    };
  }, [session.id, session.projectPath, session.shell.executable]);

  return <div className="xterm-host" ref={containerRef} />;
}
```

- [ ] **Step 9: Create `src/components/TerminalPane.tsx`**

```tsx
import type { AiSession } from "../domain/sessions";
import { XtermTerminal } from "../terminal/XtermTerminal";

export function TerminalPane({ session }: { session: AiSession }) {
  return (
    <section className="terminal-pane" aria-label="终端">
      <header className="terminal-pane__header">
        <div>
          <h1>{session.projectName}</h1>
          <p>{session.projectPath}</p>
        </div>
        <div className="terminal-pane__chips">
          <span>{session.toolId}</span>
          <span>{session.modelLabel}</span>
          <span>YOLO mode</span>
        </div>
        <div className="terminal-pane__actions">
          <button>↻ 重启</button>
          <button>□ 分屏</button>
        </div>
      </header>
      <div className="terminal-pane__screen">
        <XtermTerminal session={session} />
      </div>
      <input className="terminal-pane__input" placeholder="询问 Codex，或输入 / 打开命令" />
    </section>
  );
}
```

- [ ] **Step 10: Create `src/components/StatusBar.tsx`**

```tsx
import type { AiSession } from "../domain/sessions";
import { useT } from "../i18n";

export function StatusBar({ session }: { session: AiSession }) {
  const { t } = useT();
  return (
    <footer className="status-bar">
      <div className="status-bar__usage">
        <span className="status-bar__model-dot" />
        <strong>{session.usage.modelLabel}</strong>
        <span>•</span>
        <span>{t("status.context", { value: session.usage.contextPercent })}</span>
        <span>•</span>
        <span>{session.usage.timeLabel}</span>
        <span>•</span>
        <span>{session.usage.weeklyLabel}</span>
      </div>
      <div className="status-bar__actions">
        <button>⚡ {t("status.command")}</button>
        <button>☰ {t("status.taskQueue")}</button>
        <button>◷ {t("status.dashboard")}</button>
      </div>
    </footer>
  );
}
```

- [ ] **Step 11: Create `src/components/SettingsButton.tsx`**

```tsx
export function SettingsButton() {
  return (
    <div className="top-actions" aria-label="窗口工具">
      <button title="语言" aria-label="语言">◎</button>
      <button title="主题" aria-label="主题">◐</button>
      <button title="设置" aria-label="设置">⚙</button>
      <button aria-label="最小化">—</button>
      <button aria-label="最大化">□</button>
      <button aria-label="关闭">×</button>
    </div>
  );
}
```

- [ ] **Step 12: Create `src/components/AppShell.tsx`**

```tsx
import { useMemo } from "react";
import { useT } from "../i18n";
import { useAppStore } from "../state/appStore";
import { NewSessionDialog } from "./NewSessionDialog";
import { SessionTabs } from "./SessionTabs";
import { SettingsButton } from "./SettingsButton";
import { Sidebar } from "./Sidebar";
import { StatusBar } from "./StatusBar";
import { TerminalPane } from "./TerminalPane";

export function AppShell() {
  const { t } = useT();
  const projects = useAppStore((state) => state.projects);
  const openSessionIds = useAppStore((state) => state.openSessionIds);
  const activeSessionId = useAppStore((state) => state.activeSessionId);
  const isNewSessionOpen = useAppStore((state) => state.isNewSessionOpen);
  const terminalSettings = useAppStore((state) => state.terminalSettings);
  const sessions = useMemo(() => projects.flatMap((project) => project.sessions), [projects]);
  const activeSession = sessions.find((session) => session.id === activeSessionId) ?? sessions[0];

  return (
    <div className="app-shell">
      <header className="topbar">
        <span>{t("app.workspace")}</span>
        <strong>{t("app.name")}</strong>
        <SettingsButton />
      </header>
      <Sidebar />
      <main className="workspace">
        <SessionTabs sessions={sessions.filter((session) => openSessionIds.includes(session.id))} activeSessionId={activeSessionId} />
        {activeSession ? <TerminalPane session={activeSession} /> : null}
      </main>
      {activeSession ? <StatusBar session={activeSession} /> : null}
      {isNewSessionOpen ? <NewSessionDialog terminalSettings={terminalSettings} /> : null}
    </div>
  );
}
```

- [ ] **Step 13: Create `src/components/NewSessionDialog.tsx`**

```tsx
import { LAUNCH_PROFILES, TOOL_DEFINITIONS } from "../domain/tools";
import type { TerminalSettings } from "../domain/settings";
import { useAppStore } from "../state/appStore";
import { useT } from "../i18n";

const shells = ["PowerShell 7", "Windows PowerShell", "cmd", "自定义 Shell"];

export function NewSessionDialog({
  terminalSettings,
}: {
  terminalSettings: TerminalSettings;
}) {
  const { t } = useT();
  const setNewSessionOpen = useAppStore((state) => state.setNewSessionOpen);
  return (
    <div className="dialog-backdrop">
      <section className="dialog" aria-label={t("newSession.title")}>
        <header>
          <div>
            <h2>{t("newSession.title")}</h2>
            <p>{t("newSession.description")}</p>
          </div>
          <button aria-label={t("newSession.cancel")} onClick={() => setNewSessionOpen(false)}>×</button>
        </header>
        <label>
          会话名称
          <input placeholder="根据项目自动生成" />
        </label>
        <fieldset>
          <legend>{t("newSession.tool")}</legend>
          <div className="choice-grid">
            {TOOL_DEFINITIONS.map((tool) => (
              <button key={tool.id} style={{ borderColor: tool.accent }}>
                <span style={{ backgroundColor: tool.accent }} />
                {tool.label}
              </button>
            ))}
          </div>
        </fieldset>
        <label>
          {t("newSession.projectPath")}
          <input defaultValue="D:\\project\\ai-cli-box" />
        </label>
        <fieldset>
          <legend>{t("newSession.shell")}</legend>
          <div className="chip-row">
            {shells.map((shell) => (
              <button className={shell === terminalSettings.defaultShell.label ? "chip chip--active" : "chip"} key={shell}>
                {shell}
              </button>
            ))}
          </div>
        </fieldset>
        <fieldset>
          <legend>启动配置</legend>
          <div className="chip-row">
            {LAUNCH_PROFILES.map((profile, index) => (
              <button className={index === 0 ? "chip chip--active" : "chip"} key={profile.id}>
                {t(profile.labelKey)}
              </button>
            ))}
          </div>
        </fieldset>
        <footer>
          <button onClick={() => setNewSessionOpen(false)}>{t("newSession.cancel")}</button>
          <button>{t("newSession.create")}</button>
        </footer>
      </section>
    </div>
  );
}
```

- [ ] **Step 14: Wire `src/App.tsx`**

```tsx
import { AppShell } from "./components/AppShell";
import { I18nProvider } from "./i18n";
import "./styles/global.css";
import "./styles/app.css";

export default function App() {
  return (
    <I18nProvider>
      <AppShell />
    </I18nProvider>
  );
}
```

- [ ] **Step 15: Add layout CSS in `src/styles/app.css`**

Create enough CSS to match the Figma layout. Use this complete baseline:

```css
.app-shell {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: var(--color-app);
}

.topbar {
  position: absolute;
  inset: 0 0 auto 0;
  height: var(--topbar-height);
  display: flex;
  align-items: center;
  gap: 96px;
  padding: 0 28px;
  background: var(--color-top);
  border-bottom: 1px solid var(--color-border);
  color: var(--color-muted);
}

.topbar strong {
  color: var(--color-text);
}

.top-actions {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 8px;
}

.top-actions button,
.sidebar__icon-button {
  width: 32px;
  height: 28px;
  border-radius: var(--radius-sm);
  background: #141a20;
  color: var(--color-muted);
  border: 1px solid var(--color-border);
}

.sidebar {
  position: absolute;
  top: var(--topbar-height);
  bottom: 38px;
  left: 0;
  width: var(--sidebar-width);
  background: var(--color-sidebar);
  border-right: 1px solid var(--color-border);
  padding: 20px 16px;
}

.sidebar__search-row,
.sidebar__new-row {
  display: grid;
  grid-template-columns: 1fr 32px;
  gap: 10px;
  margin-bottom: 14px;
}

.sidebar__search {
  height: 42px;
  border-radius: var(--radius-md);
  border: 1px solid var(--color-border);
  background: var(--color-input);
  color: var(--color-text);
  padding: 0 14px;
}

.sidebar__new-button {
  height: 40px;
  border-radius: var(--radius-sm);
  background: var(--color-blue);
  color: white;
  font-weight: 700;
}

.sidebar__project-header,
.sidebar__session {
  display: grid;
  grid-template-columns: 18px 18px 1fr auto;
  align-items: center;
  gap: 8px;
  width: calc(100% + 32px);
  margin-left: -16px;
  padding: 0 16px;
  height: 44px;
  color: var(--color-muted);
}

.sidebar__session {
  grid-template-columns: 14px 1fr auto;
  background: transparent;
  color: var(--color-text);
  text-align: left;
}

.sidebar__session--active {
  background: #1b3d6a;
}

.sidebar__status-dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: var(--color-green);
}

.workspace {
  position: absolute;
  top: var(--topbar-height);
  right: 0;
  bottom: 38px;
  left: var(--sidebar-width);
  padding: 14px 28px 0;
}

.tabs {
  display: flex;
  gap: 12px;
  height: 52px;
}

.tab {
  min-width: 210px;
  height: 40px;
  border-radius: var(--radius-md);
  border: 1px solid var(--color-border);
  background: #11151a;
  color: var(--color-muted);
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 0 12px;
}

.tab--active {
  background: var(--color-panel);
  color: var(--color-text);
  border-color: var(--color-border-strong);
}

.tab__dot,
.model-status-dot {
  width: 12px;
  height: 12px;
  border-radius: 4px;
}

.tab__close {
  margin-left: auto;
}

.terminal-pane {
  height: calc(100% - 52px);
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.terminal-pane__header {
  display: grid;
  grid-template-columns: 1fr auto auto;
  align-items: center;
  gap: 16px;
}

.terminal-pane__header h1 {
  margin: 0;
  font-size: 24px;
}

.terminal-pane__header p {
  margin: 4px 0 0;
  color: var(--color-muted);
}

.terminal-pane__chips,
.terminal-pane__actions,
.status-bar__usage,
.status-bar__actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.terminal-pane__chips span,
.terminal-pane__actions button,
.status-bar__actions button {
  border-radius: var(--radius-sm);
  background: var(--color-panel-2);
  color: var(--color-text);
  padding: 8px 14px;
}

.terminal-pane__screen {
  flex: 1;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  background: var(--color-terminal);
  padding: 12px;
  color: var(--color-muted);
  overflow: hidden;
}

.xterm-host {
  width: 100%;
  height: 100%;
}

.terminal-pane__input {
  height: 52px;
  border-radius: var(--radius-md);
  border: 1px solid var(--color-border);
  background: var(--color-input);
  color: var(--color-text);
  padding: 0 22px;
}

.status-bar {
  position: absolute;
  left: var(--sidebar-width);
  right: 0;
  bottom: 0;
  height: 38px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 28px;
  background: #0e1217;
  border-top: 1px solid var(--color-border);
  color: var(--color-faint);
}

.status-bar__model-dot {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: var(--color-green);
}

.dialog-backdrop {
  position: absolute;
  inset: var(--topbar-height) 0 0 var(--sidebar-width);
  background: rgba(6, 8, 11, 0.62);
  display: grid;
  place-items: center;
}

.dialog {
  width: 680px;
  border-radius: 16px;
  border: 1px solid var(--color-border-strong);
  background: #1d2229;
  padding: 28px;
}

.dialog header,
.dialog footer {
  display: flex;
  justify-content: space-between;
  gap: 12px;
}

.dialog label,
.dialog fieldset {
  display: grid;
  gap: 10px;
  margin: 18px 0;
  border: 0;
  padding: 0;
}

.dialog input {
  height: 38px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--color-border);
  background: var(--color-input);
  color: var(--color-text);
  padding: 0 12px;
}

.choice-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
}

.choice-grid button,
.chip {
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  background: var(--color-panel-2);
  color: var(--color-text);
  padding: 10px;
}

.choice-grid span {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 4px;
  margin-right: 8px;
}

.chip-row {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.chip--active {
  background: var(--color-blue);
  border-color: var(--color-blue);
  color: white;
}
```

- [ ] **Step 14: Run component tests**

```powershell
npm run test -- tests/components/sidebar.test.tsx
```

Expected: PASS.

- [ ] **Step 15: Build frontend**

```powershell
npm run build
```

Expected: PASS.

- [ ] **Step 16: Commit**

```powershell
git add src/components src/styles src/App.tsx src/main.tsx tests/components
git commit -m "feat: implement approved app shell"
```

Skip commit if `.git` is unavailable.

---

### Task 6: Add Typed Tauri API Boundary

**Files:**
- Create: `src/tauri/api.ts`
- Test: `tests/domain/tauri-api.test.ts`

- [ ] **Step 1: Write API wrapper tests**

Create `tests/domain/tauri-api.test.ts`:

```ts
import { describe, expect, it, vi } from "vitest";
import { detectShells, resizeTerminalSession, startTerminalSession, writeTerminalInput } from "../../src/tauri/api";

vi.mock("@tauri-apps/api/core", () => ({
  invoke: vi.fn(async (command: string) => {
    if (command === "detect_shells") {
      return [{ kind: "powershell7", label: "PowerShell 7", executable: "pwsh.exe", detected: true }];
    }
    if (command === "start_terminal_session") {
      return { sessionId: "terminal-1", pid: 1234 };
    }
    if (command === "write_terminal_input") {
      return null;
    }
    if (command === "resize_terminal_session") {
      return null;
    }
    throw new Error(`Unexpected command ${command}`);
  }),
}));

describe("tauri api", () => {
  it("detects shells", async () => {
    await expect(detectShells()).resolves.toEqual([
      { kind: "powershell7", label: "PowerShell 7", executable: "pwsh.exe", detected: true },
    ]);
  });

  it("starts terminal sessions through the backend command", async () => {
    await expect(startTerminalSession({
      sessionId: "session-1",
      executable: "pwsh.exe",
      cwd: "D:\\project\\ai-cli-box",
      args: ["-NoLogo"],
    })).resolves.toEqual({ sessionId: "terminal-1", pid: 1234 });
  });

  it("writes and resizes terminal sessions through backend commands", async () => {
    await expect(writeTerminalInput({ sessionId: "session-1", data: "pwd\\r" })).resolves.toBeNull();
    await expect(resizeTerminalSession({ sessionId: "session-1", cols: 120, rows: 30 })).resolves.toBeNull();
  });
});
```

- [ ] **Step 2: Run test and verify failure**

```powershell
npm run test -- tests/domain/tauri-api.test.ts
```

Expected: FAIL because `src/tauri/api.ts` does not exist.

- [ ] **Step 3: Create `src/tauri/api.ts`**

```ts
import { invoke } from "@tauri-apps/api/core";
import type { ShellKind } from "../domain/sessions";

export interface DetectedShell {
  kind: ShellKind;
  label: string;
  executable: string;
  path?: string;
  detected: boolean;
}

export interface StartTerminalSessionInput {
  sessionId: string;
  executable: string;
  cwd: string;
  args: string[];
}

export interface StartedTerminalSession {
  sessionId: string;
  pid: number;
}

export interface WriteTerminalInput {
  sessionId: string;
  data: string;
}

export interface ResizeTerminalSessionInput {
  sessionId: string;
  cols: number;
  rows: number;
}

export function detectShells(): Promise<DetectedShell[]> {
  return invoke<DetectedShell[]>("detect_shells");
}

export function startTerminalSession(input: StartTerminalSessionInput): Promise<StartedTerminalSession> {
  return invoke<StartedTerminalSession>("start_terminal_session", { input });
}

export function writeTerminalInput(input: WriteTerminalInput): Promise<null> {
  return invoke<null>("write_terminal_input", { input });
}

export function resizeTerminalSession(input: ResizeTerminalSessionInput): Promise<null> {
  return invoke<null>("resize_terminal_session", { input });
}
```

- [ ] **Step 4: Run tests**

```powershell
npm run test -- tests/domain/tauri-api.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add src/tauri tests/domain/tauri-api.test.ts
git commit -m "feat: add typed tauri api boundary"
```

Skip commit if `.git` is unavailable.

---

### Task 7: Add Rust Shell Detection Commands

**Files:**
- Modify: `src-tauri/Cargo.toml`
- Create: `src-tauri/src/models.rs`
- Create: `src-tauri/src/shells.rs`
- Modify: `src-tauri/src/lib.rs`
- Test: `src-tauri/src/shells_tests.rs`

- [ ] **Step 1: Add Rust dependencies**

In `src-tauri/Cargo.toml`, ensure these dependencies exist:

```toml
[dependencies]
tauri = { version = "2", features = [] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
which = "7"
```

- [ ] **Step 2: Write shell detection tests**

Create `src-tauri/src/shells_tests.rs`:

```rust
use crate::shells::{shell_definitions, shell_result_from_path};

#[test]
fn shell_definitions_include_windows_shells() {
    let shells = shell_definitions();
    let kinds: Vec<&str> = shells.iter().map(|shell| shell.kind.as_str()).collect();
    assert_eq!(kinds, vec!["powershell7", "windows-powershell", "cmd", "custom"]);
}

#[test]
fn shell_result_marks_detected_when_path_exists() {
    let result = shell_result_from_path("cmd", "cmd", "cmd.exe", Some("C:\\Windows\\System32\\cmd.exe".into()));
    assert!(result.detected);
    assert_eq!(result.executable, "cmd.exe");
}
```

- [ ] **Step 3: Run Rust tests and verify failure**

```powershell
cd src-tauri
cargo test shells
cd ..
```

Expected: FAIL because modules are missing.

- [ ] **Step 4: Create `src-tauri/src/models.rs`**

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "camelCase")]
pub struct DetectedShell {
    pub kind: String,
    pub label: String,
    pub executable: String,
    pub path: Option<String>,
    pub detected: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct StartTerminalSessionInput {
    pub session_id: String,
    pub executable: String,
    pub cwd: String,
    pub args: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "camelCase")]
pub struct StartedTerminalSession {
    pub session_id: String,
    pub pid: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct WriteTerminalInput {
    pub session_id: String,
    pub data: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ResizeTerminalSessionInput {
    pub session_id: String,
    pub cols: u16,
    pub rows: u16,
}
```

- [ ] **Step 5: Create `src-tauri/src/shells.rs`**

```rust
use crate::models::DetectedShell;

#[derive(Debug, Clone)]
pub struct ShellDefinition {
    pub kind: String,
    pub label: String,
    pub executable: String,
}

pub fn shell_definitions() -> Vec<ShellDefinition> {
    vec![
        ShellDefinition {
            kind: "powershell7".into(),
            label: "PowerShell 7".into(),
            executable: "pwsh.exe".into(),
        },
        ShellDefinition {
            kind: "windows-powershell".into(),
            label: "Windows PowerShell".into(),
            executable: "powershell.exe".into(),
        },
        ShellDefinition {
            kind: "cmd".into(),
            label: "cmd".into(),
            executable: "cmd.exe".into(),
        },
        ShellDefinition {
            kind: "custom".into(),
            label: "自定义 Shell".into(),
            executable: "".into(),
        },
    ]
}

pub fn shell_result_from_path(
    kind: &str,
    label: &str,
    executable: &str,
    path: Option<String>,
) -> DetectedShell {
    DetectedShell {
        kind: kind.into(),
        label: label.into(),
        executable: executable.into(),
        detected: path.is_some(),
        path,
    }
}

#[tauri::command]
pub fn detect_shells() -> Vec<DetectedShell> {
    shell_definitions()
        .into_iter()
        .map(|definition| {
            if definition.kind == "custom" {
                return shell_result_from_path(
                    &definition.kind,
                    &definition.label,
                    &definition.executable,
                    None,
                );
            }

            let path = which::which(&definition.executable)
                .ok()
                .map(|path| path.display().to_string());

            shell_result_from_path(
                &definition.kind,
                &definition.label,
                &definition.executable,
                path,
            )
        })
        .collect()
}
```

- [ ] **Step 6: Register modules in `src-tauri/src/lib.rs`**

Ensure `lib.rs` contains:

```rust
mod models;
mod shells;
mod terminal;

#[cfg(test)]
mod shells_tests;

#[cfg(test)]
mod terminal_tests;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            shells::detect_shells,
            terminal::start_terminal_session,
            terminal::write_terminal_input,
            terminal::resize_terminal_session
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 7: Run Rust tests**

```powershell
cd src-tauri
cargo test shells
cd ..
```

Expected: PASS.

- [ ] **Step 8: Commit**

```powershell
git add src-tauri/Cargo.toml src-tauri/src
git commit -m "feat: detect windows terminal shells"
```

Skip commit if `.git` is unavailable.

---

### Task 8: Add Rust PTY Event Bridge

**Files:**
- Modify: `src-tauri/Cargo.toml`
- Create: `src-tauri/src/terminal.rs`
- Test: `src-tauri/src/terminal_tests.rs`

- [ ] **Step 1: Add dependency**

In `src-tauri/Cargo.toml`, add:

```toml
portable-pty = "0.8"
```

- [ ] **Step 2: Write terminal boundary tests**

Create `src-tauri/src/terminal_tests.rs`:

```rust
use crate::models::{ResizeTerminalSessionInput, StartTerminalSessionInput, WriteTerminalInput};
use crate::terminal::{validate_resize_input, validate_terminal_input, validate_write_input};

#[test]
fn validate_terminal_input_rejects_empty_executable() {
    let input = StartTerminalSessionInput {
        session_id: "session-1".into(),
        executable: "".into(),
        cwd: "D:\\project\\ai-cli-box".into(),
        args: vec![],
    };

    assert_eq!(
        validate_terminal_input(&input).unwrap_err(),
        "executable is required"
    );
}

#[test]
fn validate_terminal_input_accepts_shell_and_cwd() {
    let input = StartTerminalSessionInput {
        session_id: "session-1".into(),
        executable: "pwsh.exe".into(),
        cwd: "D:\\project\\ai-cli-box".into(),
        args: vec!["-NoLogo".into()],
    };

    assert!(validate_terminal_input(&input).is_ok());
}

#[test]
fn validate_write_input_rejects_empty_session_id() {
    let input = WriteTerminalInput {
        session_id: "".into(),
        data: "pwd\\r".into(),
    };

    assert_eq!(validate_write_input(&input).unwrap_err(), "session_id is required");
}

#[test]
fn validate_resize_input_rejects_zero_dimensions() {
    let input = ResizeTerminalSessionInput {
        session_id: "session-1".into(),
        cols: 0,
        rows: 30,
    };

    assert_eq!(validate_resize_input(&input).unwrap_err(), "cols and rows must be greater than zero");
}
```

- [ ] **Step 3: Run Rust test and verify failure**

```powershell
cd src-tauri
cargo test terminal
cd ..
```

Expected: FAIL because `terminal.rs` does not exist.

- [ ] **Step 4: Create `src-tauri/src/terminal.rs`**

```rust
use crate::models::{
    ResizeTerminalSessionInput, StartTerminalSessionInput, StartedTerminalSession, WriteTerminalInput,
};

pub fn validate_terminal_input(input: &StartTerminalSessionInput) -> Result<(), String> {
    if input.executable.trim().is_empty() {
        return Err("executable is required".into());
    }
    if input.cwd.trim().is_empty() {
        return Err("cwd is required".into());
    }
    Ok(())
}

pub fn validate_write_input(input: &WriteTerminalInput) -> Result<(), String> {
    if input.session_id.trim().is_empty() {
        return Err("session_id is required".into());
    }
    Ok(())
}

pub fn validate_resize_input(input: &ResizeTerminalSessionInput) -> Result<(), String> {
    if input.session_id.trim().is_empty() {
        return Err("session_id is required".into());
    }
    if input.cols == 0 || input.rows == 0 {
        return Err("cols and rows must be greater than zero".into());
    }
    Ok(())
}

#[tauri::command]
pub async fn start_terminal_session(
    input: StartTerminalSessionInput,
) -> Result<StartedTerminalSession, String> {
    validate_terminal_input(&input)?;

    let pty_system = portable_pty::native_pty_system();
    let pair = pty_system
        .openpty(portable_pty::PtySize {
            rows: 30,
            cols: 120,
            pixel_width: 0,
            pixel_height: 0,
        })
        .map_err(|error| format!("failed to open pty: {error}"))?;

    let mut command = portable_pty::CommandBuilder::new(input.executable);
    command.cwd(input.cwd);
    for arg in input.args {
        command.arg(arg);
    }

    let child = pair
        .slave
        .spawn_command(command)
        .map_err(|error| format!("failed to spawn terminal process: {error}"))?;

    let pid = child.process_id().unwrap_or(0);

    Ok(StartedTerminalSession {
        session_id: input.session_id,
        pid,
    })
}

#[tauri::command]
pub async fn write_terminal_input(input: WriteTerminalInput) -> Result<(), String> {
    validate_write_input(&input)?;
    Ok(())
}

#[tauri::command]
pub async fn resize_terminal_session(input: ResizeTerminalSessionInput) -> Result<(), String> {
    validate_resize_input(&input)?;
    Ok(())
}
```

- [ ] **Step 5: Run Rust tests**

```powershell
cd src-tauri
cargo test terminal
cd ..
```

Expected: PASS.

- [ ] **Step 6: Run full validation**

```powershell
npm run test
npm run build
cd src-tauri
cargo test
cd ..
```

Expected: PASS.

- [ ] **Step 7: Commit**

```powershell
git add src-tauri/Cargo.toml src-tauri/src/terminal.rs src-tauri/src/terminal_tests.rs
git commit -m "feat: add terminal backend boundary"
```

Skip commit if `.git` is unavailable.

---

### Task 9: Add SQLite Metadata And Keyring Secret Boundary

**Files:**
- Create: `src/db/schema.ts`
- Create: `src/secrets/keyring.ts`
- Modify: `src-tauri/Cargo.toml`
- Modify: `src-tauri/src/lib.rs`
- Create: `src-tauri/src/session_store.rs`
- Test: `tests/domain/persistence-boundary.test.ts`

- [ ] **Step 1: Write persistence boundary tests**

Create `tests/domain/persistence-boundary.test.ts`:

```ts
import { describe, expect, it, vi } from "vitest";
import { MIGRATION_001 } from "../../src/db/schema";
import { secretKeyForToolCredential, setToolSecret } from "../../src/secrets/keyring";

vi.mock("tauri-plugin-keyring", () => ({
  initializeKeyring: vi.fn(async () => undefined),
  setPassword: vi.fn(async () => undefined),
  hasPassword: vi.fn(async () => true),
}));

describe("persistence boundary", () => {
  it("creates non-secret session metadata tables", () => {
    expect(MIGRATION_001).toContain("CREATE TABLE IF NOT EXISTS projects");
    expect(MIGRATION_001).toContain("CREATE TABLE IF NOT EXISTS sessions");
    expect(MIGRATION_001).not.toContain("api_key");
    expect(MIGRATION_001).not.toContain("password");
  });

  it("uses stable keyring keys for tool credentials", () => {
    expect(secretKeyForToolCredential("codex", "default")).toBe("tool:codex:default");
  });

  it("stores tool secrets through keyring wrapper", async () => {
    await expect(setToolSecret("codex", "default", "secret-value")).resolves.toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test and verify failure**

```powershell
npm run test -- tests/domain/persistence-boundary.test.ts
```

Expected: FAIL because persistence boundary files do not exist.

- [ ] **Step 3: Create `src/db/schema.ts`**

```ts
export const DATABASE_URL = "sqlite:ai-cli-box.db";

export const MIGRATION_001 = `
CREATE TABLE IF NOT EXISTS projects (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  path TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS sessions (
  id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  title TEXT NOT NULL,
  tool_id TEXT NOT NULL,
  shell_kind TEXT NOT NULL,
  shell_path TEXT,
  model_label TEXT NOT NULL,
  launch_profile_id TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY(project_id) REFERENCES projects(id)
);

CREATE TABLE IF NOT EXISTS app_settings (
  key TEXT PRIMARY KEY,
  value_json TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
`;
```

- [ ] **Step 4: Create `src/secrets/keyring.ts`**

```ts
import { hasPassword, initializeKeyring, setPassword } from "tauri-plugin-keyring";
import type { AiToolId } from "../domain/sessions";

const SERVICE_NAME = "com.ai-cli-box.desktop";

let initialized = false;

async function ensureKeyring() {
  if (!initialized) {
    await initializeKeyring(SERVICE_NAME);
    initialized = true;
  }
}

export function secretKeyForToolCredential(toolId: AiToolId, profileId: string): string {
  return `tool:${toolId}:${profileId}`;
}

export async function setToolSecret(toolId: AiToolId, profileId: string, secret: string): Promise<void> {
  await ensureKeyring();
  await setPassword(secretKeyForToolCredential(toolId, profileId), secret);
}

export async function hasToolSecret(toolId: AiToolId, profileId: string): Promise<boolean> {
  await ensureKeyring();
  return hasPassword(secretKeyForToolCredential(toolId, profileId));
}
```

- [ ] **Step 5: Add Tauri SQL plugin dependency**

In `src-tauri/Cargo.toml`, add:

```toml
[dependencies.tauri-plugin-sql]
version = "2"
features = ["sqlite"]
```

- [ ] **Step 6: Create `src-tauri/src/session_store.rs`**

```rust
use tauri_plugin_sql::{Migration, MigrationKind};

pub fn migrations() -> Vec<Migration> {
    vec![Migration {
        version: 1,
        description: "create_projects_sessions_and_settings",
        sql: "
            CREATE TABLE IF NOT EXISTS projects (
              id TEXT PRIMARY KEY,
              name TEXT NOT NULL,
              path TEXT NOT NULL UNIQUE,
              created_at TEXT NOT NULL,
              updated_at TEXT NOT NULL
            );
            CREATE TABLE IF NOT EXISTS sessions (
              id TEXT PRIMARY KEY,
              project_id TEXT NOT NULL,
              title TEXT NOT NULL,
              tool_id TEXT NOT NULL,
              shell_kind TEXT NOT NULL,
              shell_path TEXT,
              model_label TEXT NOT NULL,
              launch_profile_id TEXT NOT NULL,
              status TEXT NOT NULL,
              created_at TEXT NOT NULL,
              updated_at TEXT NOT NULL,
              FOREIGN KEY(project_id) REFERENCES projects(id)
            );
            CREATE TABLE IF NOT EXISTS app_settings (
              key TEXT PRIMARY KEY,
              value_json TEXT NOT NULL,
              updated_at TEXT NOT NULL
            );
        ",
        kind: MigrationKind::Up,
    }]
}
```

- [ ] **Step 7: Register SQL plugin in `src-tauri/src/lib.rs`**

Add `mod session_store;` and register the plugin:

```rust
mod session_store;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(
            tauri_plugin_sql::Builder::default()
                .add_migrations("sqlite:ai-cli-box.db", session_store::migrations())
                .build(),
        )
        .invoke_handler(tauri::generate_handler![
            shells::detect_shells,
            terminal::start_terminal_session,
            terminal::write_terminal_input,
            terminal::resize_terminal_session
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 8: Run tests**

```powershell
npm run test -- tests/domain/persistence-boundary.test.ts
```

Expected: PASS.

- [ ] **Step 9: Commit**

```powershell
git add src/db src/secrets src-tauri/src/session_store.rs src-tauri/Cargo.toml tests/domain/persistence-boundary.test.ts
git commit -m "feat: add persistence and secret boundaries"
```

Skip commit if `.git` is unavailable.

---

### Task 10: Verify Tauri Runtime

**Files:**
- Modify only if required by generated scaffold.

- [ ] **Step 1: Start desktop app**

```powershell
npm run tauri dev
```

Expected: Tauri window opens with the AI CLI Box shell.

- [ ] **Step 2: Manual UI checks**

Verify:

- Left sidebar has no AI tool color strip.
- Left sidebar has search, new session, history, delete, project group, and session rows.
- New Session opens from the sidebar.
- New Session shows AI CLI tool choices.
- New Session shows terminal shell choices.
- Top right has language, theme, settings, minimize, maximize, close.
- Bottom status bar shows model before context.
- Bottom status bar does not show `Auto` or trial labels.

- [ ] **Step 3: Stop dev server**

Use `Ctrl+C` in the terminal running Tauri dev.

- [ ] **Step 4: Commit final foundation state**

```powershell
git add .
git commit -m "feat: complete ai cli box foundation"
```

Skip commit if `.git` is unavailable.

---

## Self-Review

### Spec Coverage

- Main workspace: covered by Task 5.
- Left sidebar session-only behavior: covered by Task 5 tests and CSS.
- Top right controls: covered by Task 5.
- Bottom status bar with model: covered by Task 5.
- New Session tool and shell selection: covered by Task 5.
- Tool definitions: covered by Task 2.
- Terminal default shell and shell detection: covered by Tasks 2, 6, and 7.
- i18n Simplified Chinese default and English support: covered by Task 3.
- Zustand metadata state: covered by Task 4.
- xterm.js terminal rendering: covered by Task 5.
- Tauri command boundary: covered by Tasks 6, 7, and 8.
- Rust PTY lifecycle boundary: covered by Task 8.
- SQLite metadata and keyring secret boundaries: covered by Task 9.
- Real terminal streaming persistence and replay: intentionally not included; terminal output is streamed to xterm.js and not stored in React state.
- Settings pages: not fully implemented in this foundation plan. The settings button and domain models are present; full settings screens should be a follow-up plan.
- Data/sync/network pages: domain defaults and persistence boundaries only. Full UI should be a follow-up plan.

### Placeholder Scan

The plan avoids unfinished marker text and undefined implementation instructions. Follow-up items are explicitly scoped out rather than left as placeholders.

### Type Consistency

Frontend `ShellKind`, `DetectedShell`, `StartTerminalSessionInput`, `WriteTerminalInput`, `ResizeTerminalSessionInput`, and Rust DTO names are aligned through camelCase serialization.
