# Plan: Copilot Sidebar IDE for Obsidian

Port the embedded-terminal Claude sidebar to **GitHub Copilot CLI**, keeping every feature
(tab naming, bell alerts, session-bound tab groups, per-tab conversation persistence, live
IDE context). This is the **real terminal embedded in the sidebar** — *not* Copilot's `--acp`
mode and *not* its "IDE attach" mode (the user finds that UX poor).

## Strategy

Fork the existing [`obsidian-claude-sidebar-ide`](https://github.com/coletaylor788/obsidian-claude-sidebar-ide).
It already embeds an xterm.js terminal driven by a Python PTY bridge, is multi-backend
(Claude/Codex/Gemini/OpenCode), and already implements every feature we want against Claude
Code. The job is to **add Copilot as a backend and remap three Claude-specific integrations
to Copilot's native equivalents.** The terminal, session-groups, settings, and UI layers are
backend-agnostic and carry over essentially unchanged.

**Headline finding from local investigation:** Copilot CLI is a *better* fit than Claude for
this design. Two of the hardest, hackiest parts of the Claude version become trivial, and the
IDE bridge — **now proven end-to-end against a real Copilot** — works exactly like Claude's
(injected selection/file nudges, not model-pull tools).

## Feature parity map

| Feature | Claude mechanism (today) | Copilot mechanism (verified locally) | Effort |
|---|---|---|---|
| Embedded terminal | Python PTY → `claude` | Python PTY → `copilot` (just a new backend entry) | Trivial |
| Per-tab conversation persistence | Scrape `~/.claude/projects/*.jsonl` to *capture* the id Claude assigns (a whole module + 12 tests) | **Mint a UUID and pass `copilot --session-id=<uuid>`; resume with `copilot --resume=<uuid>`.** No disk scraping. ✅ verified: minting + resume both work | **Much simpler** |
| Tab naming (`/rename`) | Parse `custom-title` lines out of the conversation `.jsonl` | Read the `name:` field from `~/.copilot/session-state/<uuid>/workspace.yaml` (we own the uuid). Optional: live title from Copilot's OSC terminal-title stream | **Simpler** |
| Bell / "needs attention" | `.claude` Stop/Notification **hooks** POST to the plugin | Copilot **also has hooks** (`stop`/`notification`, `.github/hooks` + user-level `config.json`). Same pattern; PTY-BEL sniffing as fallback | Medium |
| **Live file/selection context + diff review** | WebSocket MCP server + lock file in `~/.claude/ide/`; `--ide` + `CLAUDE_CODE_SSE_PORT` | **Same model, proven working.** Plugin runs an IDE MCP server + writes a lock to `~/.copilot/ide/`; Copilot auto-connects and treats `getCurrentSelection`/`getOpenEditors`/`openDiff` as **native IDE features (injected nudges)**, not model tools. Transport differs (Unix socket + Streamable-HTTP MCP, not TCP WebSocket) | Medium — **de-risked** |
| Session-bound tab groups | Pure Obsidian workspace logic, backend-independent | Unchanged | None |
| Shift+Enter multiline, YOLO, folder targeting | Backend flags/config | Copilot: `--allow-all-tools`/`--allow-all`, `/terminal-setup`, `-C <dir>` | Trivial |

## What gets deleted vs. added

- **Delete:** `claude-session-capture.ts` (+ its tests) — replaced by one `crypto.randomUUID()`
  call. This removes the most fragile, race-prone code in the project (cwd-encoding, mtime
  races, cross-tab dedupe).
- **Add:** a `copilot` backend definition, a tiny `workspace.yaml` title reader, Copilot hook
  scripts, and a `~/.copilot/ide/` IDE MCP server (the existing one, re-transported to a Unix
  socket + Streamable-HTTP MCP — see the IDE section).

## Phases (each independently testable)

1. **Fork & rebrand** — clone the repo, rename plugin id/name to `copilot-sidebar-ide`,
   strip Claude-only naming, get a clean `bun run check` + load in Obsidian.
2. **Copilot backend + spawn/resume** — add the backend; `copilot` runs in a tab; verify
   `--continue` and the build pipeline. *(Core terminal usable.)*
3. **Per-tab persistence** — mint a UUID per tab, spawn `--session-id=<uuid>`, resume
   `--resume=<uuid>` on reload. Delete the capture module.
4. **Tab titles** — read `workspace.yaml` `name`; refresh on focus; persist via `getState`.
5. **Bell / notifications** — install Copilot `stop`/`notification` hooks that ping the plugin
   with the tab id; light the per-tab bell. (PTY-BEL fallback if hook payload lacks tab id.)
6. **IDE context (proven; the real work)** — port the existing `ide-server.ts` from Claude's
   TCP-WebSocket transport to Copilot's **Unix-socket + Streamable-HTTP MCP** transport, write the
   lock to `~/.copilot/ide/` with Copilot's schema. Copilot then auto-connects and consumes
   `getCurrentSelection`/`getOpenEditors`/`openDiff` as native IDE features (injected nudges +
   diff approval), exactly like Claude. Reuse the selection-tracking, tool catalog, and `DiffModal`
   as-is.
7. **Polish** — settings copy, README/docs, CI release workflow, version reset to `0.1.0`.
   (Sprites.dev mobile path deferred — see Decisions.)

## IDE context: does Copilot learn my open file & selection, "just like Claude"? — Yes (proven)

I drove a **real Copilot** against a hand-built IDE server and captured the whole handshake. It
works, and it works the *right* way — push-based IDE nudges, not model-pull tools. Evidence:

- Copilot's log: `IDE lock file change detected` → `Found matching IDE for workspace: Obsidian`
  → **`Connected to IDE MCP server: Obsidian`**. Its TUI shows **`● Connected to Obsidian`**.
- Copilot then logs `Skipping tool getCurrentSelection for client ide` (and the same for
  `getLatestSelection`, `getOpenEditors`, `getWorkspaceFolders`, `openDiff`). Translation: it
  **recognizes the standard VS Code IDE tool vocabulary and wires it into its own IDE features**
  (selection/file awareness, diff approval) rather than exposing them as model tools. That is
  precisely the "injected nudge" behavior you asked for — the same as Claude Code.

**How it differs from Claude (transport only):** Claude uses a TCP WebSocket (`ws://127.0.0.1:port`,
header `x-claude-code-ide-authorization`). Copilot uses a **Unix domain socket + Streamable-HTTP
MCP**: the plugin listens on a Unix socket, and Copilot does `POST /mcp` (request/response) plus a
long-lived `GET /mcp` SSE stream for server→client push. Auth is whatever header(s) you declare in
the lock. The plugin's MCP message handling, tool catalog, selection push, and diff modal are
otherwise unchanged.

**Lock file** (`~/.copilot/ide/<id>.lock`) — required fields confirmed by Copilot's schema
validator: `socketPath`, `scheme`, `headers` (object — the auth headers Copilot will send back),
`timestamp`; plus `workspaceFolders` + `ideName` for matching. Full schema + a working server
sketch are in the appendix.

> Track B (a generic `--additional-mcp-config` MCP server) is **not** the plan — it's pull-based and
> can't do injected nudges. We're doing the real IDE integration above.

## Decisions (open questions closed)

- **Bell / notifications — solved, low risk.** Use Copilot's `stop`/`notification` hooks (it has a
  hooks system) to ping the plugin with the tab id. If the hook payload can't carry a tab id, fall
  back to sniffing the PTY stream for BEL (`\x07`) / OSC-9 — guaranteed since we own the stream. Bell
  works either way.
- **IDE integration — proven, no longer a risk.** Confirmed end-to-end against a real Copilot (see
  above). Remaining work is implementation (port the transport), not investigation. The only detail
  to pin during coding is the exact selection-delivery method name on the SSE push channel, which the
  plugin's own server logs reveal immediately.
- **Multi-account — no special handling for v1.** Spawned tabs inherit Copilot's selected account;
  document the `COPILOT_GITHUB_TOKEN` / `GH_HOST` env overrides for users who need a specific account.
- **Sprites.dev mobile — deferred.** Ship desktop-only for v1.

## Appendix

Detailed file-by-file changes, exact commands, verified evidence, the lock-file/hook schemas,
and the spike procedure live in [`docs/appendix.md`](docs/appendix.md).
