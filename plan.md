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
IDE bridge uses the same lock-file protocol family Claude does.

## Feature parity map

| Feature | Claude mechanism (today) | Copilot mechanism (verified locally) | Effort |
|---|---|---|---|
| Embedded terminal | Python PTY → `claude` | Python PTY → `copilot` (just a new backend entry) | Trivial |
| Per-tab conversation persistence | Scrape `~/.claude/projects/*.jsonl` to *capture* the id Claude assigns (a whole module + 12 tests) | **Mint a UUID and pass `copilot --session-id=<uuid>`; resume with `copilot --resume=<uuid>`.** No disk scraping. ✅ verified: minting + resume both work | **Much simpler** |
| Tab naming (`/rename`) | Parse `custom-title` lines out of the conversation `.jsonl` | Read the `name:` field from `~/.copilot/session-state/<uuid>/workspace.yaml` (we own the uuid). Optional: live title from Copilot's OSC terminal-title stream | **Simpler** |
| Bell / "needs attention" | `.claude` Stop/Notification **hooks** POST to the plugin | Copilot **also has hooks** (`stop`/`notification`, `.github/hooks` + user-level `config.json`). Same pattern; PTY-BEL sniffing as fallback | Medium |
| Live file/selection context + diff review | WebSocket MCP server + lock file in `~/.claude/ide/`; `--ide` + `CLAUDE_CODE_SSE_PORT` | Copilot watches `~/.copilot/ide/` for lock files and auto-connects (`ide.autoConnect`, `/ide`, `ide.openDiffOnEdit`). Reuse the existing server; change lock dir + confirm auth handshake | Medium (1 spike) |
| Session-bound tab groups | Pure Obsidian workspace logic, backend-independent | Unchanged | None |
| Shift+Enter multiline, YOLO, folder targeting | Backend flags/config | Copilot: `--allow-all-tools`/`--allow-all`, `/terminal-setup`, `-C <dir>` | Trivial |

## What gets deleted vs. added

- **Delete:** `claude-session-capture.ts` (+ its tests) — replaced by one `crypto.randomUUID()`
  call. This removes the most fragile, race-prone code in the project (cwd-encoding, mtime
  races, cross-tab dedupe).
- **Add:** a `copilot` backend definition, a tiny `workspace.yaml` title reader, Copilot hook
  scripts, and a `~/.copilot/ide/` lock writer (a 1-line path change from the existing one,
  pending the handshake spike).

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
6. **IDE context bridge (the one spike)** — point the lock writer at `~/.copilot/ide/`,
   confirm Copilot's exact lock schema + auth header by capturing a live `/ide` handshake,
   then reuse the existing selection-push + diff-review server.
7. **Polish** — settings copy, README/docs, CI release workflow, version reset to `0.1.0`.
   Decide whether to keep or defer the Sprites.dev mobile path (see Open questions).

## Risks & open questions

- **IDE handshake details (only real unknown).** Copilot definitely watches `~/.copilot/ide/`
  and auto-connects, but my offline probe was inconclusive because this machine has
  `ide.autoConnect:false`. Phase 6 must capture one real connection to confirm the lock-file
  field names, the auth header (Claude uses `x-claude-code-ide-authorization`), and the MCP
  `initialize`/`tools` shape. Everything else in the server is reusable as-is.
- **Hook event names/payload.** Confirm Copilot's exact `stop`/`notification` event keys and
  whether the payload can carry a per-tab id (env-injected) for precise bell targeting.
- **Multi-account.** This machine has multiple logged-in GitHub users; confirm spawned tabs
  inherit the intended account.
- **Sprites mobile mode.** The Claude fork provisions a cloud VM for mobile. Recommend
  **deferring** it for v1 and shipping desktop-only, since Copilot's session/IDE model differs.

## Appendix

Detailed file-by-file changes, exact commands, verified evidence, the lock-file/hook schemas,
and the spike procedure live in [`docs/appendix.md`](docs/appendix.md).
