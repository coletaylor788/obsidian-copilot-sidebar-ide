# Appendix — Unified Agent Sidebar: technical detail

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
  - `ide.autoConnect` (default `true`) — watches `~/.copilot/ide/` for lock files and auto-connects. **Proven:** with it enabled, a real Copilot detected our lock, validated its schema, matched by workspace + `ideName`, and connected (`Connected to IDE MCP server: Obsidian`). See §7 for the full captured protocol.
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

## 2. Capability-adapter architecture & file map

Introduce a `BackendAdapter` per agent, registered by id, composing four optional capability slots
plus a spawn builder. The core (terminal, session-groups, view, settings) depends only on these
interfaces — no `backend === "claude"` branching.

```ts
// src/backends/adapter.ts
export interface BackendAdapter {
  id: string;                  // "claude" | "copilot" | "codex" | ...
  label: string;
  binary: string;
  buildSpawnCommand(ctx: SpawnCtx): { cliCmd: string; env: Record<string, string> };
  persistence?: SessionPersistence;   // undefined → generic --continue only
  title?: TitleSource;                // undefined → static label
  notifications?: NotificationHooks;  // undefined → BEL/OSC sniff fallback only
  ide?: IdeBridgeFactory;             // undefined → no IDE integration
}

export interface SessionPersistence {
  prepare(ctx): { sessionId?: string };          // before spawn (Copilot: mint UUID)
  onSpawned?(view, cwd): void;                    // after spawn (Claude: start capture poll)
  resumeArgs(sessionId: string | null): string;  // "--resume <id>" | "--continue" | ...
}
export interface TitleSource { read(sessionId: string, cwd: string): string | null; }
export interface NotificationHooks { install(cwd: string, tabId: string): { dispose(): void }; }
export interface IdeBridge { start(): void; stop(): void; pushSelection(): void; }
export interface IdeBridgeFactory { create(app, getVaultPath): IdeBridge; }
```

A registry `BACKENDS: Record<string, BackendAdapter>` replaces today's data-only `CLI_BACKENDS`.
The **shared** IDE pieces (`getToolCatalog`, `handleToolCall`, selection tracking, `DiffModal`) move
to `src/ide/` and are reused by both transports — only the transport + lock differ per backend.

### File / module change map (unified — evolving the existing repo)

| File / module | Action |
|---|---|
| `src/backends/` (new dir) | `adapter.ts` (interfaces) + `claude.ts`, `copilot.ts`, `codex.ts`, … each registering an adapter; replaces data-only `backends.ts` |
| `src/types.ts` | Rename `claudeSessionId`/`claudeSessionTitle` → `agentSessionId`/`sessionTitle` |
| `src/claude-session-capture.ts` (+ test) | **Keep** — becomes the Claude `SessionPersistence` impl (not deleted; just no longer Copilot's concern) |
| `new: src/copilot-session.ts` (+ test) | Copilot `SessionPersistence` (mint UUID) + `TitleSource` (read `workspace.yaml` `name`) |
| `src/ide/` (was `ide-server.ts`/`ide-tools.ts`/`diff-modal.ts`) | Split transport from core: shared catalog/selection/diff + `IdeTransport`s `claude-ws.ts` and `copilot-http-mcp.ts` |
| `src/terminal-view.ts` | Replace `=== "claude"` gates with `adapter.persistence?`/`adapter.title?`; rename state to `agentSessionId` |
| `src/shell-manager.ts` / `remote-shell-manager.ts` | Delegate command building to `adapter.buildSpawnCommand()`; hooks via `adapter.notifications?` |
| `src/main.ts` | Generic `notifyCallback`; start `adapter.ide?` bridge instead of the Claude-only IDE server |
| `src/settings.ts` | Backend dropdown already exists; add Copilot; agent-neutral copy |
| `manifest.json`, `package.json` | Agent-neutral id/name/description; **keep version lineage** (bump minor, don't reset to 0.1.0) |

---

## 3. Backend definition (drop-in)

```ts
// src/backends/copilot.ts (adapter base data)
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

- **New tab:** `copilot --session-id <uuid> [--allow-all-tools] [flags]`, where `uuid = crypto.randomUUID()` is minted by the Copilot `SessionPersistence.prepare()` and stored on the view (persisted by the existing `getState()` plumbing).
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

## 7. IDE context bridge — PROVEN protocol (port `ide-server.ts` to this transport)

**Goal:** the running Copilot knows the file open in Obsidian and the text selected — delivered as
injected IDE nudges, like Claude (not model-pull tools). **Status: confirmed end-to-end** by driving
a real `copilot 1.0.64` against a hand-built server.

### What was observed (evidence)

From the spawned Copilot's own log against our test server:
```
IDE lock file change detected: <id>.lock
Found matching IDE for workspace: Obsidian (PID …)
Connected to IDE MCP server: Obsidian
Skipping tool getCurrentSelection for client ide
Skipping tool getLatestSelection for client ide
Skipping tool getOpenEditors  for client ide
Skipping tool getWorkspaceFolders for client ide
Skipping tool openDiff for client ide
```
TUI showed `● Connected to Obsidian` / `1 MCP server`. The `Skipping tool … for client ide` lines
mean Copilot recognizes the standard VS Code IDE tool surface and consumes it as IDE features
(injected selection/file context + diff approval) instead of registering them as model tools — the
Claude-style behavior.

### Transport (the only real difference from Claude)

| | Claude | **Copilot** |
|---|---|---|
| Discovery | lock in `~/.claude/ide/<port>.lock` | lock in `~/.copilot/ide/<id>.lock` |
| Transport | TCP WebSocket `ws://127.0.0.1:<port>` | **Unix domain socket** at `socketPath` |
| Wire | WebSocket frames, MCP JSON-RPC | **Streamable-HTTP MCP**: `POST /mcp` (req/resp) + `GET /mcp` SSE (server→client push) |
| Auth | fixed header `x-claude-code-ide-authorization` = `authToken` | **arbitrary header(s) you declare in the lock's `headers`**, echoed on every request |
| Discovery trigger | env `CLAUDE_CODE_SSE_PORT` + dir scan | `ide.autoConnect` (default true) watches `~/.copilot/ide/`, matches by `workspaceFolders` + `ideName` |

### Lock file schema (validated by Copilot)

Required (Copilot rejects the lock otherwise — discovered via its Zod validation error):
`socketPath` (string), `scheme` (string), `headers` (object), `timestamp` (number). Plus
`workspaceFolders` (array — must contain Copilot's cwd) and `ideName` for matching. Working example:

```json
{
  "socketPath": "/tmp/obsidian-ide-<rand>.sock",
  "scheme": "ws",
  "headers": { "x-obsidian-ide-auth": "<random-token>" },
  "timestamp": 1781976000000,
  "workspaceFolders": ["/Users/you/vault"],
  "ideName": "Obsidian",
  "pid": 12345,
  "transport": "ws"
}
```

`headers` is the auth mechanism: whatever you put there, Copilot sends back on every `POST /mcp`
and on the `GET /mcp` SSE request (verified — our `x-obsidian-ide-auth` came back on every call).

### Handshake observed (what the server must implement)

1. `POST /mcp` → `{"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"copilot-cli","version":"1.0.64-1"}}}` → respond with `{result:{protocolVersion, serverInfo, capabilities:{tools:{}}}}` and an `Mcp-Session-Id` response header.
2. `POST /mcp` → `notifications/initialized` (respond `202`).
3. `GET /mcp` with `accept: text/event-stream`, `mcp-session-id`, `mcp-protocol-version` → keep open; this is the **server→client push channel** for selection/file nudges.
4. `POST /mcp` → `tools/list` → return the IDE tool catalog (reuse `getToolCatalog()` verbatim:
   `getCurrentSelection`, `getLatestSelection`, `getOpenEditors`, `getWorkspaceFolders`, `openDiff`,
   `getDiagnostics`, `openFile`, `saveDocument`, …).

### Port plan for `ide-server.ts`

- Replace the TCP `http.createServer().listen(port)` + WS-upgrade machinery with: `http.createServer()`
  listening on a **Unix socket path**, handling `POST /mcp` (JSON-RPC req/resp), `GET /mcp` (SSE push),
  `DELETE /mcp` (teardown). Drop the hand-rolled WS framing (`ws-framing.ts`) — Streamable-HTTP needs no
  WebSocket frames.
- Keep verbatim: `getToolCatalog()` / `handleToolCall()` (`ide-tools.ts`), the selection tracking +
  debounced push (`pushSelection`), and the `DiffModal` wiring.
- Push selection/file changes as JSON-RPC notifications over the open `GET /mcp` SSE stream (the
  analog of Claude's `selection_changed`). **Confirm the exact notification method name Copilot
  ingests during Phase 6** — trivial, because our server logs every byte Copilot sends/accepts; try
  `selection_changed` first and watch whether Copilot also calls `getCurrentSelection` internally.
- Lock writer: emit the schema above into `~/.copilot/ide/<id>.lock`; keep the stale-lock cleanup
  (match on `ideName === "Obsidian"` + live `pid`).
- No `--ide` flag or `CLAUDE_CODE_SSE_PORT` needed; Copilot auto-connects when `ide.autoConnect` is
  on (default). Optionally surface a one-time settings note if the user disabled it.

### Reference: minimal working server (captured during the spike)

A ~70-line Node server (Unix socket + `POST`/`GET /mcp`) was enough to make Copilot print
`Connected to Obsidian` and fetch the tool catalog. The production version is the existing
`ide-server.ts` with the transport swapped per above.

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
