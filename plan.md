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
| **Live file/selection context + diff review** | WebSocket MCP server + lock file in `~/.claude/ide/`; `--ide` + `CLAUDE_CODE_SSE_PORT` | **Two-track (see below).** Track B (local MCP server via `--additional-mcp-config`) is guaranteed; Track A (reuse the lock-file WS server at `~/.copilot/ide/`) adds Claude-style live push + native diff if the handshake matches | Medium — **the one real risk** |
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
6. **IDE context (two tracks)** — ship **Track B** first: a local MCP server registered via
   `copilot --additional-mcp-config` exposing `get_active_note`/`get_selection`/`open_note`/
   `propose_edit` (guaranteed live awareness of the open file + selection). Then add **Track A**:
   reuse `ide-server.ts` at `~/.copilot/ide/` for Claude-style live push + native diff, once the
   handshake is confirmed against the official extension's lock.
7. **Polish** — settings copy, README/docs, CI release workflow, version reset to `0.1.0`.
   (Sprites.dev mobile path deferred — see Decisions.)

## IDE context: does Copilot learn my open file & selection, "just like Claude"?

**Yes — that's exactly what Copilot's IDE integration is for**, and it's the right vehicle. Confirmed
from `copilot help config` and Copilot's own logs: it watches `~/.copilot/ide/` for lock files,
auto-connects (`ide.autoConnect`), exposes `/ide`, routes edit diffs to the IDE for approval
(`ide.openDiffOnEdit`), and *"diagnostics and selection features still work."* Same capability class
as Claude's IDE bridge.

**Honesty check:** this is the **only** feature I could not yet prove end-to-end with Obsidian acting
as the server. A local spike (hand-rolled lock + WS server mirroring Claude's exact handshake) did not
get Copilot to connect — but that test was inconclusive (the PTY harness didn't boot Copilot cleanly,
and Copilot's handshake may differ from Claude's). So I'm treating it as **promising but unproven**,
not "done." To guarantee the user gets the capability regardless, the plan uses two tracks:

- **Track B — local MCP server (guaranteed, ship first).** The plugin runs a tiny MCP server and
  registers it on the spawned process with `copilot --additional-mcp-config <json>`, exposing tools
  like `get_active_note`, `get_selection`, `open_note`, `propose_edit`. Copilot's first-class MCP
  support means it can *pull* the current Obsidian file/selection on demand and write edits back. No
  reverse-engineering required. Trade-off vs Claude: pull-based (no automatic `selection_changed`
  push, and diff approval happens in-terminal unless we add a modal).
- **Track A — lock-file WS bridge (best UX, add once confirmed).** Reuse the existing `ide-server.ts`,
  pointed at `~/.copilot/ide/`. If Copilot's handshake matches, this gives Claude-style **live
  selection push** + **native side-by-side diff approval**. De-risk by capturing what the *official*
  Copilot IDE integration (VS Code extension) writes to `~/.copilot/ide/` and mirroring it exactly,
  rather than guessing.

Net: the terminal will know your open file and selection like Claude — guaranteed via Track B, and
with the nicer live-push/diff UX via Track A when the handshake is confirmed.

## Decisions (open questions closed)

- **Bell / notifications — solved, low risk.** Use Copilot's `stop`/`notification` hooks (it has a
  hooks system) to ping the plugin with the tab id. If the hook payload can't carry a tab id, fall
  back to sniffing the PTY stream for BEL (`\x07`) / OSC-9 — guaranteed since we own the stream. Bell
  works either way.
- **Multi-account — no special handling for v1.** Spawned tabs inherit Copilot's selected account;
  document the `COPILOT_GITHUB_TOKEN` / `GH_HOST` env overrides for users who need a specific
  account. (This machine has multiple logged-in users; that's fine.)
- **Sprites.dev mobile — deferred.** Ship desktop-only for v1. Copilot's session/IDE model differs
  enough that the cloud-VM path should be revisited separately, not block the first release.
- **Remaining real risk: Track A handshake only**, and it's fully covered by the Track B guarantee.

## Appendix

Detailed file-by-file changes, exact commands, verified evidence, the lock-file/hook schemas,
and the spike procedure live in [`docs/appendix.md`](docs/appendix.md).
