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
  - `ide.autoConnect` (default `true`) — *"watch for new IDE lock files"*, auto-connect on startup. This machine had it set to `false`. **Update:** I temporarily set it to `true` and re-ran the lock-file probe; Copilot still didn't connect to a hand-rolled (Claude-schema) server, but the PTY harness didn't boot Copilot's TUI cleanly that run, so the result is inconclusive — see §7.
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

## 7. IDE context bridge — two tracks

**Goal:** the running Copilot must know the file open in Obsidian and the text selected, like Claude.
This is the only feature not yet proven end-to-end with Obsidian as the server, so the design uses a
guaranteed track plus a best-UX track.

### Spike result so far (honest status)

- ✅ Copilot *has* the capability: watches `~/.copilot/ide/`, `ide.autoConnect`, `/ide`,
  `ide.openDiffOnEdit`, *"diagnostics and selection features still work."*
- ⚠️ A local test (permissive WS server + lock mirroring Claude's `{pid, workspaceFolders, ideName,
  transport, authToken}` schema, `ide.autoConnect` temporarily enabled, `/ide` issued) **did not
  produce a connection**. Inconclusive: the PTY harness failed to boot Copilot's TUI cleanly that
  run, and Copilot's handshake/lock schema may differ from Claude's. Not evidence it can't work — but
  enough that we don't assume the Claude server is a drop-in.

### Track B — local MCP server (guaranteed; implement first)

Copilot has first-class MCP support (`~/.copilot/mcp-config.json`, `copilot mcp add`, and
`--additional-mcp-config <json>` to augment per-session). The plugin runs a small stdio/HTTP MCP
server and registers it on each spawned tab:

```
copilot --session-id <uuid> --additional-mcp-config @/path/to/obsidian-mcp.json [...]
```

Tools to expose:
- `get_active_note` → `{ path, content }` of the note focused in Obsidian's main editor.
- `get_selection` → `{ text, path, range }` of the current selection (reuse the same
  `getEditorFromRecentLeaf` logic already in `ide-tools.ts`).
- `list_open_notes` → open tabs.
- `open_note` → open a note in Obsidian.
- `propose_edit` → show the existing `DiffModal` for accept/reject, then apply.

Pros: no protocol reverse-engineering; works today. Cons: pull-based (model calls the tool; no
automatic `selection_changed` push), and diff approval uses our modal rather than Copilot's native
IDE diff. Good enough to fully satisfy "the terminal knows my open file/selection."

### Track A — lock-file WS bridge (best UX; add once confirmed)

Reuse `ide-server.ts` almost verbatim, pointed at `~/.copilot/ide/`. If Copilot's handshake matches,
this yields Claude-parity: debounced `selection_changed` push + native diff-on-edit approval.

Required confirmations (do this by capturing the **official** integration, not by guessing):
1. Install/enable the official Copilot IDE integration in VS Code, open a workspace, and inspect the
   lock file it writes to `~/.copilot/ide/` → that's the exact schema/fields to emit.
2. Capture the WS upgrade Copilot sends (auth header name — Claude uses
   `x-claude-code-ide-authorization`) and the MCP `initialize`/`tools/list` shape.
3. Adjust `ide-server.ts`: lock dir, lock fields, auth header, `serverInfo`. Keep the WS framing,
   tool catalog, selection push, and `DiffModal` wiring.

### Recommendation

Ship Track B in Phase 6 (reliable parity for "knows my file/selection"); pursue Track A as a
fast-follow once the official-extension capture confirms the handshake.

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
