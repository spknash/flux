# Flux — project overview

This document describes the **current** state of the repository (what exists in code today) and how the app behaves when you run it. Update this file when the architecture or major features change so it stays a reliable map of the project.

---

## What this project is

**Flux** is an Electron desktop app (product name “Flux”, npm package `flux`) aimed at task-driven AI agent orchestration. The long-term vision is described in `README.md` (kanban, agent sessions, git worktrees, planning assistant). **This file reflects what is implemented in the codebase right now**, which is a subset of that vision.

---

## Tech stack

| Layer | Choice |
|--------|--------|
| Desktop shell | Electron 41 (Electron Forge 7 + Vite plugin) |
| UI | React 18, TypeScript, Tailwind CSS 3 |
| Drag-and-drop | `@hello-pangea/dnd` |
| Package manager | pnpm (`pnpm-lock.yaml` present) |

Scripts (from `package.json`): `pnpm start` (Electron Forge dev), `pnpm package` / `pnpm make` (distribution), `pnpm lint` (ESLint for `.ts` / `.tsx`).

---

## Repository layout (high level)

- **`src/main.ts`** — Main process: window creation, IPC handlers for tasks, macOS title bar / theme tweaks.
- **`src/main/TaskStore.ts`** — Persists tasks as JSON under the Electron user data directory (`tasks.json`), with atomic save via temp file + rename.
- **`src/preload.ts`** — Exposes `window.electronAPI` (platform + typed task CRUD) through `contextBridge`.
- **`src/renderer.tsx`** — React entry; mounts `App`.
- **`src/App.tsx`** — Application state and routing between sidebar “views”.
- **`src/components/`** — UI: shell, sidebar, board, columns, cards, new-task modal, top bar, agent badge.
- **`src/types.ts`** — `Task`, `TaskStatus`, `Agent`, column and agent metadata.
- **`src/seed.ts`** — Static demo tasks used as initial UI data.
- **`forge.config.ts`** — Forge + Vite build targets (main, preload, renderer).

---

## How the app works today

### Processes and IPC

- Standard **Electron** split: **main** (Node) and **renderer** (Chromium + React).
- **Context isolation** is on; **Node integration** is off in the browser window.
- The **preload** script defines `window.electronAPI` with:
  - `platform` — `process.platform` (used for macOS drag region vs. other OS).
  - `tasks.getAll`, `tasks.create`, `tasks.update`, `tasks.delete` — backed by `ipcMain.handle` in `main.ts` and `TaskStore`.

### Task model

- **`TaskStatus`**: `backlog` | `in-progress` | `needs-input` | `done`.
- **`Agent`**: `claude-code` | `codex` | `cursor` (labels only in the UI; no agent processes are launched).
- Each task has: `id`, `title`, `status`, `agent`, `createdAt` (ISO string).

### Renderer behavior (source of truth in the UI)

- **`App.tsx` holds tasks in React state**, initialized from **`SEED_TASKS`** in `src/seed.ts` — not loaded from `TaskStore` on startup.
- **Dragging** tasks between columns updates **only React state** (`handleDragEnd`).
- **Creating** a task (modal from the Backlog column) adds a client-generated task to state.
- **Deleting** a task removes it from state.
- The **Plan** sidebar entry shows a placeholder: *“Planning assistant coming soon”* (no planning UI or API calls).

So: **persistence and IPC for tasks exist in main/preload**, but the **current board UI does not sync with the persisted file** unless future work wires `electronAPI.tasks` into `App` lifecycle.

### Main process / persistence

- On startup, `TaskStore.init()` loads `tasks.json` if present; malformed files fall back to an empty list with a console warning.
- IPC handlers perform CRUD and write through `TaskStore.save()` (atomic write on disk).

### Packaging

- Electron Forge is configured with Squirrel, ZIP (darwin), RPM, Deb makers and the Vite plugin for main/preload/renderer builds.

---

## Implemented vs. not yet in code

**Present today**

- Electron window (dev server or packaged HTML), dark-themed UI, macOS-friendly chrome where applicable.
- Kanban-style board with four columns, drag-and-drop, per-task agent badge, delete on card, “new task” from Backlog with title + agent.
- Sidebar switching between **Board** and **Plan** (placeholder).
- JSON-backed task store in main process + preload API (not driving the board yet).

**Not implemented in this repo (vs. README vision)**

- SQLite or services named SessionManager, WorktreeService, PlanningService, NotificationService (README describes a fuller architecture).
- Spawning agent CLIs, terminals, `node-pty`, or git worktrees.
- Planning assistant that calls external APIs.
- Desktop notifications for task events.
- Loading/saving the board from `electronAPI.tasks` end-to-end.

When adding features, update this section and the “How the app works today” section so they stay accurate.

---

## Maintenance

- **Canonical product narrative**: see `README.md`.
- **Living snapshot of the codebase**: this file (`project-overview.md`).
