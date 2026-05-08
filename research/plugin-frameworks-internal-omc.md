# Internal source brief — oh-my-claudecode (OMC) anatomy

> Raw research output. Cite file paths from this brief verbatim in the Korean report.

## 1. Plugin packaging

### `.claude-plugin/plugin.json` (root manifest)
- Required field: only `name`. All other metadata optional.
- OMC fields: `name`, `version` (4.13.2, synced from package.json), `description`, `author`, `repository`, `homepage`, `license`, `keywords`, plus extension pointers `"skills": "./skills/"` and `"mcpServers": "./.mcp.json"`.
- Anthropic's full schema (per official docs) supports many more pointers: `commands`, `agents`, `hooks`, `lspServers`, `outputStyles`, `experimental.themes`, `experimental.monitors`, `userConfig`, `channels`, `dependencies`.

### `.claude-plugin/marketplace.json`
- Self-hosted marketplace metadata. `$schema: https://anthropic.com/claude-code/marketplace.schema.json`.
- OMC ships exactly one plugin in `plugins[]` with `source: "./"`, category `productivity`.

### `.mcp.json` (MCP server registration)
```json
{
  "mcpServers": {
    "t": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/bridge/mcp-server.cjs"]
    }
  }
}
```
- `${CLAUDE_PLUGIN_ROOT}` is expanded by Claude Code at runtime to the installed plugin path.
- Server alias `t` → tools surface as `mcp__plugin_oh-my-claudecode_t__*`.

### `package.json`
- Bin: `omc`, `omc-cli`, `oh-my-claudecode` → `bridge/cli.cjs`.
- Files whitelist (what npm publishes): `dist`, `agents`, `bridge`, `commands`, `hooks`, `scripts`, `skills`, `templates`, `docs`, `.claude-plugin`, `.mcp.json`, `README.md`, `LICENSE`.
- Build chain: `tsc → build-skill-bridge → build-mcp-server → build-bridge-entry → compose-docs → build-runtime-cli → build-team-server → build-cli`.

## 2. Skills layer

### Frontmatter schema (OMC convention)
```yaml
---
name: <skill-id>
description: <one-line summary>
argument-hint: "<optional usage hint>"
level: 1 | 2 | 3 | 4
---
<Purpose>...</Purpose>
<Use_When>...</Use_When>
<Do_Not_Use_When>...</Do_Not_Use_When>
<Steps>...</Steps>
```

### Sample: `skills/ralph/SKILL.md` (level 4)
```yaml
---
name: ralph
description: Self-referential loop until task completion with configurable verification reviewer
argument-hint: "[--no-deslop] [--critic=architect|critic|codex] <task description>"
level: 4
---
```
- PRD-driven: scaffolds `.omc/prd.json`, refines acceptance criteria, iterates story-by-story until reviewer-verified.
- Step 7 → 7.5 → 7.6 → 8 chain in one turn; "boulder never stops" signal blocks polite-stop anti-pattern.

### Sample: `skills/autopilot/SKILL.md` (level 4)
- Six phases: Expansion → Planning → Execution → QA → Validation → Cleanup.
- Phase 0 detects pre-existing artifacts: ralplan consensus plan or deep-interview spec → skips redundant analyst+architect work.
- Phase 4 fans out to `architect` + `security-reviewer` + `code-reviewer` in parallel; all must approve.

### Sample: `skills/cancel/SKILL.md` (level 2)
- Reads session-scoped `state_list_active` to detect active mode, then unwinds in dependency order.

### Routing
- `scripts/keyword-detector.mjs` (UserPromptSubmit, 5s timeout) matches priority-ordered keywords (cancelomc, ralph, autopilot, ultrawork/ulw, ccg, ralplan, deep interview, deslop). Writes signal to `.omc/state/sessions/{sessionId}/skill-active-state.json`.
- `scripts/skill-injector.mjs` (UserPromptSubmit, 3s timeout) recursively scans `~/.claude/skills/omc-learned/`, `~/.omc/skills/`, `.omc/skills/`, parses YAML frontmatter, injects matching skill text via `<system-reminder>`.

## 3. Agents layer

### Frontmatter schema
```yaml
---
name: <agent-id>
description: <role summary>
model: haiku | sonnet | opus
level: 1..4
disallowedTools: Write, Edit   # optional, e.g. architect is READ-ONLY
---
<Agent_Prompt>
  <Role>...</Role>
  <Why_This_Matters>...</Why_This_Matters>
  <Success_Criteria>...</Success_Criteria>
  <Constraints>...</Constraints>
  <Investigation_Protocol>...</Investigation_Protocol>
  <Tool_Usage>...</Tool_Usage>
  <Execution_Policy>...</Execution_Policy>
  <Output_Format>...</Output_Format>
  <Failure_Modes_To_Avoid>...</Failure_Modes_To_Avoid>
</Agent_Prompt>
```

### `agents/executor.md` (sonnet, level 2)
- Role: implement requested change with smallest viable diff.
- Hard rules in `<Constraints>`: work alone; don't refactor adjacent code; never modify plan files; escalate to architect after 3 failed attempts on the same issue.
- Notes orchestrators must use `wrapWithPreamble()` from `src/agents/preamble.ts` to prevent recursive sub-agent spawning.

### `agents/architect.md` (opus, level 3, disallowedTools: Write, Edit)
- READ-ONLY by tool restriction, not just prompt.
- Investigation_Protocol: gather → form hypothesis → cross-reference with file:line citations → synthesize Diagnosis/Root Cause/Recommendations. 3-failure circuit breaker.

### Lanes (from `docs/ARCHITECTURE.md`)
| Lane | Agents |
|---|---|
| Build/Analysis | explore, analyst, planner, architect, debugger, executor, verifier, tracer |
| Review | security-reviewer, code-reviewer |
| Domain | test-engineer, designer, writer, qa-tester, scientist, git-master, document-specialist, code-simplifier |
| Coordination | critic |

## 4. Hooks pipeline (`hooks/hooks.json`)

| Event | Scripts (timeout) | Purpose |
|---|---|---|
| UserPromptSubmit | keyword-detector.mjs (5s), skill-injector.mjs (3s) | Detect magic keywords, inject relevant skills |
| SessionStart | session-start.mjs (5s), project-memory-session.mjs (5s), wiki-session-start.mjs (5s); matcher `init`/`maintenance` for setup/setup-maintenance | Bootstrap project memory + wiki + setup |
| PreToolUse | pre-tool-enforcer.mjs (3s) | Validate tool perms, enforce delegation |
| PermissionRequest | permission-handler.mjs (5s for Bash) | Auto-approve safe ops |
| PostToolUse | post-tool-verifier.mjs (3s), project-memory-posttool.mjs (3s), post-tool-rules-injector.mjs (3s) | Verify output, capture patterns, inject rules |
| PostToolUseFailure | post-tool-use-failure.mjs (3s) | Record failure context |
| SubagentStart / SubagentStop | subagent-tracker.mjs, verify-deliverables.mjs | Track delegation; verify completion |
| Stop | context-guard-stop.mjs, persistent-mode.mjs, code-simplifier.mjs | Save mode state, optional cleanup |
| PreCompact | pre-compact.mjs, project-memory-precompact.mjs, wiki-pre-compact.mjs | Summarize before context reset |
| SessionEnd | session-end.mjs (30s), wiki-session-end.mjs (30s) | Archive logs |

### Injection pattern
Hooks output JSON containing `<system-reminder>`. Claude Code injects the value into the system prompt mid-conversation. Examples:
- `[MAGIC KEYWORD: autopilot]` after keyword-detector match
- `The boulder never stops` while ralph/ultrawork is active
- `<remember>...</remember>` (7-day TTL) and `<remember priority>` (permanent)

### Kill switches
`DISABLE_OMC=1`, `OMC_SKIP_HOOKS=keyword-detector,skill-injector` (comma-separated).

### Worktree safety (post-tool-rules-injector)
Project root derived from accessed file's path via `findProjectRoot`, NOT `data.cwd`. A `.git` *file* at worktree root stops upward walk. Prevents parent-repo rule leakage in worktrees.

## 5. MCP bridge (`bridge/`)

| File | Size | Role |
|---|---|---|
| `bridge/mcp-server.cjs` | 963KB | Compiled bundle: state_*, notepad_*, project_memory_*, wiki_*, lsp_*, ast_grep_*, python_repl |
| `bridge/team-mcp.cjs` | 657KB | Team MCP: TeamCreate, SendMessage, TaskCreate/List/Get/Update; inbox/outbox; heartbeats |
| `bridge/cli.cjs` | 3.14MB | `omc` CLI binary; delegates to src/cli/* |
| `bridge/runtime-cli.cjs` | 271KB | Bootstrap for tmux team workers |
| `bridge/team-bridge.cjs` | 81KB | Daemon lifecycle for team bridge |

### Tool registry (representative)
- **State**: `state_read`, `state_write`, `state_clear`, `state_list_active`, `state_get_status`
- **Notepad**: `notepad_read`, `notepad_write_priority`, `notepad_write_working`, `notepad_write_manual`, `notepad_prune`, `notepad_stats`
- **Project memory**: `project_memory_read/write/add_note/add_directive`
- **Wiki**: `wiki_add/delete/ingest/lint/list/query/read`
- **Code intel**: `lsp_*` (hover, goto_definition, find_references, diagnostics, rename, code_actions, document_symbols, workspace_symbols), `ast_grep_search/replace`
- **Trace**: `trace_summary`, `trace_timeline`
- **Shared memory**: `shared_memory_*` (cross-session)

## 6. `src/` TypeScript layout

```
src/
├── agents/        # Prompt assembly + Worker Preamble (preamble.ts)
├── autoresearch/  # Stateful research loop
├── cli/           # CLI subcommands
├── commands/      # Plugin command registration
├── config/        # JSONC parser + schema validation
├── constants/     # Names of skills/agents/modes
├── features/      # state-manager, delegation-routing, model-routing, verification (~20 modules)
├── hooks/         # 52 subdirs incl. keyword-detector, skill-injector, autopilot, ralph, team-pipeline
├── hud/           # Heads-up display
├── installer/     # Plugin install + config merge
├── interop/       # Codex / Gemini providers
├── lib/           # FS, paths, CLI parsing
├── mcp/           # omc-tools-server.ts (registry), job-management.ts, team-server.ts
├── notifications/ # Push notifications, HUD updates
├── planning/      # Plan generation
├── platform/      # Local vs remote detection
├── providers/     # Claude / Codex / Gemini bootstrap
├── ralphthon/     # Ralph iteration engine
├── shared/        # Shared types
├── skills/        # Frontmatter parser + trigger logic
├── team/          # 54 files: runtime.ts, tmux-session.ts, mcp-team-bridge.ts, team-ops.ts
├── tools/         # LSP/diagnostics/Python REPL implementations
├── types/         # TypeScript types/enums
├── utils/         # Generic utilities
├── verification/  # Verifier engine
└── __tests__/     # vitest suite
```

### Highest-leverage learning targets
- `src/mcp/omc-tools-server.ts` — minimal worked example of registering a tool.
- `src/hooks/keyword-detector/` — read stdin JSON, match priority, emit reminder.
- `src/hooks/skill-injector/` — recursive frontmatter scan + dedup.
- `src/team/runtime.ts` + `src/team/tmux-session.ts` — multi-pane worker orchestration.

## 7. State / runtime artifacts (`.omc/`)

```
.omc/
├── state/sessions/{sessionId}/{autopilot|ralph|ultrawork|ultraqa|team|ralplan|skill-active|boulder|cancel-signal}-state.json
├── state/swarm.db                # SQLite (legacy omc-teams)
├── state/team-bridge/{team}/*.heartbeat.json
├── notepad.md                    # 7-day TTL session memory
├── project-memory.json           # Repo-scoped persistent knowledge
├── plans/{autopilot-impl|ralplan-*|consensus-*}.md
├── research/                     # Autoresearch output
├── logs/{session,team}-*.log
├── specs/deep-interview-*.md
├── handoffs/{team-plan,team-prd,team-exec}.md
└── skills/                       # Project-local learned skills
```

### Team pipeline stages
`team-plan → team-prd → team-exec → team-verify → team-fix (bounded loop)`.

## 8. Build & release

- Build chain composed in package.json scripts; each `build-*.mjs` is an `esbuild` bundler producing one of the `bridge/*.cjs` artifacts.
- `scripts/compose-docs.mjs` — `{{INCLUDE:partials/...}}` template interpolation; `docs/templates/*.template.md` → `docs/*.md`; partials copied to `docs/shared/` for skill consumption.
- `scripts/sync-version.sh` — keeps `package.json`, `plugin.json`, `marketplace.json` versions aligned (run by `npm version`).
- `scripts/release.ts` — bump → build → docs → tests → tag → publish.

## 9. Behavioral patterns to reuse

- **Worker Preamble Protocol** (`src/agents/preamble.ts`, referenced from `agents/executor.md`): orchestrators wrap delegated work to forbid recursive sub-agent spawning; controls context nesting.
- **Delegation Enforcer**: agent prompts include explicit "what you are NOT responsible for"; tool restrictions (`disallowedTools`) reinforce role boundaries.
- **Magic-keyword routing**: priority-ordered keyword matching at UserPromptSubmit → state file → skill-injector hydrates context.
- **System-reminder injection**: hooks emit `<system-reminder>` JSON; Claude Code injects mid-conversation; this is the primary mechanism for runtime guidance without modifying CLAUDE.md.
- **`<remember>` tags**: short-form notepad writes (7-day TTL); `<remember priority>` permanent.
- **Kill switches**: `DISABLE_OMC`, `OMC_SKIP_HOOKS=...`.
- **Worktree-safe project root**: walk up from accessed file, stop at `.git` *file* (worktree marker).
- **Context safety**: PreCompact summarizes before reset; notepad auto-prune; state-manager rehydrates mode after compaction.
- **PRD-driven loop** (ralph): scaffold → refine → iterate per-story → reviewer-verified completion. "Boulder" reminder blocks premature stop.
- **Bounded fix loop** (autopilot QA Phase 3): max 5 cycles; same-error-repeats-3-times escapes to fundamental-issue report.
- **Multi-perspective parallel review** (autopilot Phase 4): architect + security-reviewer + code-reviewer in parallel; all-must-approve gate.
