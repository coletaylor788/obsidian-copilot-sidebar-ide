# Plan: Unified Agent Sidebar (add GitHub Copilot CLI)

> **Implementation status (in progress):** Phases 1–6 are implemented, verified, and green
> (`bun run check`: 38 tests + build) on branch `feat/unified-backend-adapters` →
> [PR #2](https://github.com/coletaylor788/obsidian-claude-sidebar-ide/pull/2) in the plugin repo.
> Phase 7 (agent-neutral rebrand) is pending a name choice. Each Copilot capability was verified
> against a real `copilot 1.0.64`: `--session-id` mint+resume, `workspace.yaml` titles, the
> Unix-socket Streamable-HTTP MCP IDE bridge (`Connected to IDE MCP server: Obsidian`), and the
> `agentStop`/`notification` hook bell.

Evolve the existing embedded-terminal sidebar into one **multi-backend** Obsidian plugin where you
pick your agent — Claude Code, **GitHub Copilot CLI**, Codex, Gemini, OpenCode — from a dropdown,
with full feature parity per backend: tab naming, bell alerts, session-bound tab groups, per-tab
conversation persistence, and live IDE context. It runs the **real CLI in an embedded terminal** —
not Copilot's `--acp` mode and not its "IDE attach" mode (poor UX).

**Decisions (this session):**
- **One unified plugin**, not a Copilot-only fork.
- **Evolve `obsidian-claude-sidebar-ide`** (keeps git history, already multi-backend) into an
  **agent-neutral** plugin. This `obsidian-copilot-sidebar-ide` repo is the design/plan home.

## Why unified (not a fork)

- The plugin is **already multi-backend** (Claude/Codex/Gemini/OpenCode share one terminal,
  session-groups, persistence, settings, and UI core). The genuinely Claude-specific branching is
  **~6 `backend === "claude"` checks**; the other ~90 "claude" references are just variable names
  (`claudeSessionId`) and the default "Claude" label.
- A fork would **duplicate thousands of lines** of shared scaffolding to isolate a few hundred lines
  of per-backend logic, then split every future fix across two repos.
- Shared features (session groups, bell, UI polish) then improve **every** backend at once.

## Architecture: per-backend capability adapters

Lift the four backend-specific behaviors behind a small adapter each backend implements; the core
stays generic. Adding a future agent = implement one adapter.

| Capability | Interface | Claude impl (exists) | Copilot impl (new — verified) |
|---|---|---|---|
| Spawn / resume | `buildSpawnCommand()` | flags + `--ide` + trust dir | `--session-id`/`--resume`, `--allow-all-tools` |
| Per-tab persistence | `SessionPersistence` | scrape `~/.claude/projects/*.jsonl` | **mint UUID + `--session-id`** |
| Tab title | `TitleSource` | `.jsonl` `custom-title` | `workspace.yaml` `name` |
| Bell / attention | `NotificationHooks` | `.claude` hooks → `/notify` | `.copilot`/`.github` hooks → `/notify` |
| IDE bridge | `IdeTransport` | TCP WebSocket, `~/.claude/ide` | **Unix socket + Streamable-HTTP MCP, `~/.copilot/ide`** |

Backends without a capability (Codex/Gemini/OpenCode) just get the terminal + generic resume.
**Shared across all IDE-capable backends:** the tool catalog (`getCurrentSelection`/`getOpenEditors`/
`openDiff`), selection tracking, and the diff modal — only the transport + lock file differ.

## Phases (each independently testable)

1. **Adapter refactor — no behavior change.** Extract the ~6 Claude branches + Claude-named state
   behind the adapter interfaces; rename `claudeSessionId`→`agentSessionId`, etc. Claude keeps
   working as the regression oracle; `bun run check` stays green. *(Pure refactor — safest first
   step, and the foundation for everything else.)*
2. **Copilot backend.** Add the `copilot` backend entry; terminal spawn + `--continue`. *(Copilot
   usable as a terminal.)*
3. **Copilot persistence adapter.** Mint a UUID per tab → `--session-id=<uuid>`; resume
   `--resume=<uuid>`. ✅ verified end-to-end.
4. **Copilot title adapter.** Read `name` from `~/.copilot/session-state/<uuid>/workspace.yaml`;
   refresh on focus; persist via `getState`.
5. **Copilot bell adapter.** Install `stop`/`notification` hooks that ping `/notify` with the tab id;
   light the per-tab bell. PTY-BEL/OSC fallback.
6. **Copilot IDE adapter.** Implement the `IdeTransport` for Copilot: Unix-socket + Streamable-HTTP
   MCP server, lock in `~/.copilot/ide/`. ✅ protocol captured. Reuse the shared tool catalog,
   selection push, and diff modal.
7. **Rebrand + ship.** Agent-neutral name/manifest/icons, settings copy, README/docs, CI release,
   version bump.

## Copilot specifics — verified locally (`copilot 1.0.64`)

- **Persistence:** `copilot --session-id=<uuid>` mints a session with that exact id; `--resume=<uuid>`
  resumes it (turns 1→2). Replaces Claude's fragile disk-scraping entirely.
- **Title:** `name:` in `~/.copilot/session-state/<uuid>/workspace.yaml` (set by `/rename`).
- **IDE (the headline):** a real Copilot connected to our hand-built server
  (`Connected to IDE MCP server: Obsidian`; TUI: `● Connected to Obsidian`) and treated
  `getCurrentSelection`/`getOpenEditors`/`openDiff` as **native IDE features — injected nudges, not
  model-pull tools** (`Skipping tool getCurrentSelection for client ide`). Transport = Unix socket +
  Streamable-HTTP MCP (`POST /mcp` + `GET /mcp` SSE push); auth via headers declared in the lock.
- **Bell:** Copilot has a hooks system (`stop`/`notification`).

## Risks & mitigations

- **Phase-1 refactor touches shared spawn/IDE wiring.** Mitigate by keeping it strictly
  behavior-preserving with Claude as the oracle (`bun run check` + a manual smoke of each existing
  backend) before adding Copilot.
- **Abstraction leak** if Copilot later diverges hard. Low today — Claude and Copilot are parallel
  enough that four small interfaces cover the differences; revisit only if a backend needs more.
- **One coding detail to pin (not a blocker):** the exact selection-push notification method on the
  SSE channel — the plugin's own server logs reveal it instantly during Phase 6.
- **Deferred:** Sprites.dev mobile (desktop-first). **Multi-account:** inherit Copilot's account;
  document `COPILOT_GITHUB_TOKEN`/`GH_HOST` overrides.

## Appendix

Adapter interfaces, file-by-file changes, exact commands, and the full verified Copilot protocol
(session-id, `workspace.yaml`, hooks, IDE Unix-socket/Streamable-HTTP MCP) are in
[`docs/appendix.md`](docs/appendix.md).
