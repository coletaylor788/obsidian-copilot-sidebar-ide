# Appendix — Copilot Sidebar IDE technical detail

Supporting detail for [`../plan.md`](../plan.md). Everything marked **✅ verified** was
confirmed on this machine against `GitHub Copilot CLI 1.0.64-1`. Items marked **🔎 spike**
need one live confirmation during implementation.

---

## 1. Evidence gathered (what we actually confirmed)

- **Copilot session store is a clean SQLite DB**, not per-cwd JSONL like Claude.
  - `~/.copilot/session-store.db` → `sessions(id, cwd, repository, branch, summary, created_at, updated_at, host_type)`, plus `turns`, `checkpoints`, `session_files`, `session_refs`.
  - `~/.copilot/session-state/<uuid>/` → `workspace.yaml`, `events.jsonl`, `checkpoints/`, `session.db`.
- **`--session-id=<uuid>` mints a session with that exact id.** ✅ verified:
  - Ran `copilot -p "Reply with exactly: OK" --session-id=<fresh-uuid> --allow-all-tools` → a row with that id appeared in `sessions`, and `session-state/<uuid>/workspace.yaml` was created.
  - Ran `copilot -p "..." --resume=<same-uuid> --allow-all-tools` → `turns` count went 1 → 2. Resume-by-id works.
- **`workspace.yaml` carries the human title.** Example:
  ```yaml
  id: 915358dd-...           # == the uuid we minted
  cwd: /private/tmp
  name: 'Reply with exactly: OK'   # auto-summary; /rename sets this and flips user_named: true
  user_named: false
  ```
- **Copilot watches the IDE lock dir per workspace.** ✅ from `~/.copilot/logs/process-*.log`:
  `Starting IDE lock file watcher for workspace: /Users/cole`.
- **IDE config knobs** (`copilot help config`):
  - `ide.autoConnect` (default `true`) — *"watch for new IDE lock files"*, auto-connect on startup. **This machine has it set to `false`** in `~/.copilot/settings.json`, which is why an offline lock-file probe did not connect.
  - `ide.openDiffOnEdit` (default `true`) — show edit diffs in the connected IDE; *"diagnostics and selection features still work."*
  - `updateTerminalTitle` (default `true`, `COPILOT_DISABLE_TERMINAL_TITLE` to disable) — emits OSC title with the agent's current intent.
  - `terminalProgress` (default `true`) — emits OSC `9;4` progress while working.
- **Copilot has a hooks system** (`copilot help config`): `hooks` keyed by event name (same
  schema as `.github/hooks/*.json`); inline user-level hooks live in `~/.copilot/config.json`;
  `disableAllHooks` master switch.
- **Relevant interactive commands** (`copilot help commands`): `/ide` (Connect to an IDE
  workspace), `/rename`, `/resume`, `/session`, `/diff`, `/terminal-setup` (shift+enter).
- **Relevant flags** (`copilot --help`): `--continue`, `-r/--resume[=id|name|prefix]`,
  `--session-id <id>`, `-n/--name`, `-p/--prompt`, `--allow-all-tools`/`--allow-all`/`--yolo`,
  `--add-dir`, `-C <dir>`, `--model`, `--banner`, `--no-auto-update`.

---

## 2. File-by-file change map (relative to the fork)

| File | Action |
|---|---|
| `manifest.json`, `package.json` | id → `copilot-sidebar-ide`, name/description/author, version → `0.1.0` |
| `src/backends.ts` | Add a `copilot` backend (below); make it the default |
| `src/types.ts` | Rename `claudeSessionId`/`claudeSessionTitle` → `copilotSessionId`/`sessionTitle` (or generalize to `agentSessionId`) |
| `src/claude-session-capture.ts` + `.test.ts` | **Delete.** Replaced by `crypto.randomUUID()` at tab creation |
| `src/terminal-view.ts` | Mint `copilotSessionId` on first spawn; read title from `workspace.yaml`; keep `getState`/`setState`, bell, `applyTabHeaderTitle` |
| `src/shell-manager.ts` | Build the Copilot command (below); swap Claude hook install for Copilot hook install; set lock/notify env |
| `src/ide-server.ts` | Change lock dir to `~/.copilot/ide/`; adjust auth header + `serverInfo` per spike; keep WS/MCP/selection/diff logic |
| `src/main.ts` | Rename Claude strings in `notifyCallback`; everything else (session groups, nav commands, selection push) unchanged |
| `src/settings.ts` | Re-label; keep backend dropdown, default cwd, auto-resume, CLI flags |
| `new: src/copilot-session.ts` | Small pure helper: read `name`/`user_named` from `session-state/<id>/workspace.yaml` (+ unit test) |

---

## 3. Backend definition (drop-in)

```ts
// src/backends.ts
copilot: {
  label: "GitHub Copilot",
  binary: "copilot",
  pathHints: ["/opt/homebrew/bin", "~/.local/bin"],
  yoloFlag: "--allow-all-tools",      // or "--allow-all" / "--yolo"
  resumeFlag: "--continue",           // fallback when no per-tab id yet
  resumeIsSubcommand: false,
  resumeByIdFlag: "--resume",         // append the minted uuid → --resume <uuid>
},
```

## 4. Spawn command construction

- **New tab:** `copilot --session-id <uuid> [--allow-all-tools] [flags]`, where `uuid = crypto.randomUUID()` is generated in `terminal-view.ts` and stored on the view (persisted by the existing `getState()` plumbing).
- **Resume on reload:** `copilot --resume <uuid> [...]`. If a tab somehow has no uuid, fall back to `--continue`.
- **cwd / folder targeting:** spawn with `cwd` (existing logic) and/or pass `-C <dir>`; pre-trust via `--add-dir <cwd>` or rely on `settings.json` `trustedFolders` to avoid a first-run prompt (Copilot equivalent of the Claude `trustedDirectories` pre-trust step).
- The PATH-resolution, Python-PTY, resize, and UTF-8 decoder logic in `shell-manager.ts` are backend-agnostic — keep them.

## 5. Tab title

We own the uuid, so the title source is a single file read:

```
~/.copilot/session-state/<uuid>/workspace.yaml  →  name:  (use only when user_named: true,
                                                           else fall back to a default label)
```

Wire it exactly like the Claude version: `refreshSessionTitle()` on `active-leaf-change`,
`applyTabHeaderTitle()` to push text into the tab header, persist `sessionTitle` via
`getState()` so reloads render instantly. **Optional enhancement:** also parse OSC `0;`/`2;`
title sequences from the PTY stream (`updateTerminalTitle` is on by default) for a live
"current intent" label between renames.

## 6. Bell / notifications (🔎 confirm event names + payload)

Reuse the hook→HTTP→plugin pattern. The plugin's `IdeServer` already exposes a `/notify`
POST endpoint and a per-tab `setNeedsAttention()`; only the hook installation changes.

- Install Copilot hooks (user-level `~/.copilot/config.json` `hooks`, or a workspace
  `.github/hooks/*.json`) for the `stop` and `notification` events. Each hook runs a small
  Node script that POSTs `{ type, tab_id }` to `http://127.0.0.1:<port>/notify`.
- Carry the tab id into the hook via an env var set at spawn (Copilot analog of
  `CLAUDE_OBSIDIAN_TAB_ID`) so the bell targets the exact tab.
- **🔎 Spike:** confirm the exact event keys (`stop`/`notification` vs other), the hook JSON
  field (`run`/`command`), and which env/substitution vars are available in the payload.
- **Fallback:** if hooks can't carry a tab id, sniff the PTY stream for BEL (`\x07`) / OSC `9`
  desktop-notification sequences in `terminal-view.ts` and raise the bell on the focused-away
  tab. (The Claude fork moved *away* from BEL sniffing to hooks for reliability; we keep it as
  a safety net.)

## 7. IDE context bridge (the spike)

The existing `ide-server.ts` is a WebSocket MCP **server** that (a) writes a lock file, (b)
answers `initialize`/`tools/list`/`tools/call`, (c) pushes debounced `selection_changed`, and
(d) drives a side-by-side diff modal for `openDiff`. Copilot is the **client** that discovers
the lock and connects — the same role Claude's CLI plays. Required changes:

- **Lock dir:** `~/.claude/ide/` → `~/.copilot/ide/`. Keep `<port>.lock` naming unless the
  spike shows otherwise.
- **🔎 Auth header:** Claude uses `x-claude-code-ide-authorization`. Confirm Copilot's header
  name and whether it reads `authToken` from the lock the same way.
- **🔎 Lock schema:** Claude writes `{ pid, workspaceFolders, ideName, transport: "ws", authToken }`. Confirm Copilot's required fields (it may want `workspaceFolders` to contain its cwd, which it already scopes the watcher to).
- **Discovery:** Copilot scans the dir (no `CLAUDE_CODE_SSE_PORT` analog appears necessary);
  confirm whether any env var or flag is needed, or just `ide.autoConnect:true` / `/ide`.

### Spike procedure (≈30 min)

1. Temporarily set `ide.autoConnect:true` in `~/.copilot/settings.json` (this machine has it off).
2. Run a tiny WS server that writes a lock to `~/.copilot/ide/<port>.lock` (`workspaceFolders=[cwd]`, random `authToken`) and **logs the HTTP upgrade request headers + first WS frames**.
3. Launch `copilot` in that cwd (or type `/ide`), and capture: the auth header name/value, the
   MCP `initialize` params, and the `tools/list` request. Adjust `ide-server.ts` to match.
4. Re-test `selection_changed` push and `openDiff` round-trip end-to-end.

---

## 8. Build / test / release (inherited, verified to exist)

```bash
bun install
bun run dev     # esbuild watch → main.js
bun run build   # production
bun run test    # bun test (pure helpers)
bun run check   # tsc --noEmit && bun test && bun run build
```

PTY scripts (`terminal_pty.py` / `terminal_win.py`) are base64-embedded at build time — no
change needed. CI release workflow (`.github/workflows/release.yml`) publishes `main.js`,
`manifest.json`, `styles.css` on `v*` tags; update names/version only.

## 9. Cleanup note

The spike/verification created one throwaway session (cwd `/private/tmp`) in
`~/.copilot/session-store.db`. Harmless; can be left or pruned via `/session`.
