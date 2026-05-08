# External source brief — large Claude Code plugin frameworks

> Primary sources only. Cite URLs verbatim in the Korean report.

## 1. obra/superpowers (Jesse Vincent)

- Repo: https://github.com/obra/superpowers (active)
- Marketplace: https://github.com/obra/superpowers-marketplace
- Community add-ons: https://github.com/obra/superpowers-skills
- Design post: https://blog.fsck.com/2025/10/09/superpowers/

### Repo layout
```
superpowers/
├── .claude-plugin/{plugin.json, marketplace.json}
├── skills/                # 14 skill dirs, each with SKILL.md only
│   ├── brainstorming/
│   ├── subagent-driven-development/
│   ├── test-driven-development/
│   ├── writing-plans/
│   ├── writing-skills/
│   ├── dispatching-parallel-agents/
│   ├── executing-plans/
│   ├── finishing-a-development-branch/
│   ├── receiving-code-review/
│   ├── requesting-code-review/
│   ├── systematic-debugging/
│   ├── using-git-worktrees/
│   ├── using-superpowers/
│   └── verification-before-completion/
├── hooks/{hooks.json, session-start (shell)}
├── CLAUDE.md, AGENTS.md
├── .codex-plugin/, .cursor-plugin/, .opencode/, gemini-extension.json
└── (no agents/, no commands/, no MCP server, no src/, no build pipeline)
```

### plugin.json (verbatim)
```json
{
  "name": "superpowers",
  "description": "Core skills library for Claude Code: TDD, debugging, collaboration patterns, and proven techniques",
  "version": "5.1.0",
  "author": { "name": "Jesse Vincent", "email": "jesse@fsck.com" },
  "homepage": "https://github.com/obra/superpowers",
  "repository": "https://github.com/obra/superpowers",
  "license": "MIT",
  "keywords": ["skills", "tdd", "debugging", "collaboration", "best-practices", "workflows"]
}
```

### Sample SKILL.md frontmatter
- `brainstorming/SKILL.md`:
```yaml
---
name: brainstorming
description: "You MUST use this before any creative work - creating features, building
  components, adding functionality, or modifying behavior. Explores user intent,
  requirements and design before implementation."
---
```
- `subagent-driven-development/SKILL.md`:
```yaml
---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---
```

### `writing-skills/SKILL.md` excerpt (testing-as-design rationale)
> "Testing revealed that when a description summarizes the skill's workflow, Claude
> may follow the description instead of reading the full skill content... When the
> description was changed to just 'Use when executing implementation plans with
> independent tasks' (no workflow summary), Claude correctly read the flowchart
> and followed the two-stage review process."

> "Writing skills IS Test-Driven Development applied to process documentation."

### hooks.json (verbatim)
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```
- A single SessionStart hook bootstraps skill loading. No PreToolUse, no PostToolUse — skills carry the behavior.

### Distribution
- Anthropic official marketplace: `/plugin install superpowers@claude-plugins-official`
- Self-hosted: `/plugin marketplace add obra/superpowers-marketplace` then `/plugin install superpowers@superpowers-marketplace`
- Multi-harness: ships parallel manifests for Codex CLI, Codex App, Gemini CLI, OpenCode, Cursor, GitHub Copilot CLI — same skill markdown, different harness adapters.

### Design philosophy (Jesse Vincent, blog post + CLAUDE.md)
- "Skills are what give your agents Superpowers."
- Self-reinforcing loop: agents help write more skills, which makes the next skill cheaper to write.
- "You can hand a model a book or a document or a codebase and say 'Read this. Think about it. Write down the new stuff you learned.'"
- Personal vs. shipped skills: personal in `~/.claude/skills`, plugin-shipped in `skills/`.
- From CLAUDE.md: "Our internal skill philosophy differs from Anthropic's published guidance on writing skills. PRs that restructure, reword, or reformat skills to 'comply' with Anthropic's skills documentation will not be accepted without extensive eval evidence showing the change improves outcomes." — i.e., skills are tuned by behavioral evals, not style guides.

## 2. SuperClaude Framework

- Repo: https://github.com/SuperClaude-Org/SuperClaude_Framework
- Version observed: 4.3.0
- Self-description: "meta-programming configuration framework that transforms Claude Code into a structured development platform through behavioral instruction injection and component orchestration."

### Counts (per README)
| Commands | Agents | Modes | MCP Servers |
|---|---|---|---|
| 30 | 20 | 7 | 8 |

### Layout
```
SuperClaude_Framework/
├── .claude/{settings.json, skills/}
├── plugins/        # Native Claude Code plugin format
├── skills/         # Standalone skill files
├── src/            # Python installer source
├── docs/user-guide/{commands.md, agents.md}
├── CLAUDE.md, AGENTS.md
└── pyproject.toml  # ships as Python package
```

### Install
```
pipx install superclaude
superclaude install
```
or `/plugin install superclaude`.

### Composition unit
- Slash commands (`/sc:implement`, `/sc:analyze`, ...) + 9 cognitive personas (architect, frontend, security, qa, ...) auto-activated by keyword.
- Behavioral injection lives in `CLAUDE.md`-level instructions installed by the Python tool.

## 3. Anthropic official plugin reference

- Plugin authoring guide: https://code.claude.com/docs/en/plugins
- Reference (manifest schema, all extension points): https://code.claude.com/docs/en/plugins-reference
- Official marketplace: https://github.com/anthropics/claude-plugins-official
- Skill spec: https://agentskills.io/specification

### Extension points (per the reference)
| Path | Type | Purpose |
|---|---|---|
| `skills/` | Skills | `SKILL.md` folders; namespaced `/<plugin>:<skill>` |
| `commands/` | Commands | Flat `.md` files (legacy; prefer skills) |
| `agents/` | Agents | Markdown with YAML frontmatter |
| `hooks/hooks.json` | Hooks | Lifecycle event handlers (~28 event types) |
| `.mcp.json` | MCP servers | External tool/service integration |
| `.lsp.json` | LSP servers | Real-time code intelligence |
| `monitors/monitors.json` | Monitors | Background process watchers (experimental) |
| `themes/` | Themes | (experimental) |
| `bin/` | Executables | Added to PATH while plugin active |

### plugin.json — full optional fields
```json
{
  "name": "...",                     // required, kebab-case
  "version": "...", "description": "...", "author": {...},
  "homepage": "...", "repository": "...", "license": "...",
  "keywords": [...],
  "skills": "./skills/",
  "commands": ["./commands/x.md"],
  "agents": ["./agents/y.md"],
  "hooks": "./hooks/hooks.json",
  "mcpServers": "./.mcp.json",
  "lspServers": "./.lsp.json",
  "outputStyles": "./styles/",
  "experimental": { "themes": "./themes/", "monitors": "./monitors.json" },
  "userConfig": { "api_token": { "type":"string","sensitive":true,"title":"...","description":"..." } },
  "channels": [ ... ],
  "dependencies": [ {"name":"helper","version":"~2.1.0"} ]
}
```
- Note `userConfig` (prompted at install), inter-plugin `dependencies`, and `channels` (message bus injection).

### Hook event types (28; abridged)
SessionStart, Setup, UserPromptSubmit, UserPromptExpansion, PreToolUse, PermissionRequest, PermissionDenied, PostToolUse, PostToolUseFailure, PostToolBatch, Notification, SubagentStart, SubagentStop, TaskCreated, TaskCompleted, Stop, StopFailure, TeammateIdle, InstructionsLoaded, ConfigChange, CwdChanged, FileChanged, WorktreeCreate, WorktreeRemove, PreCompact, PostCompact, Elicitation, ElicitationResult, SessionEnd.

### Hook action types
`command`, `http`, `mcp_tool`, `prompt`, `agent`.

### Standalone (`.claude/`) vs plugin
| | Standalone | Plugin |
|---|---|---|
| Skill name | `/hello` | `/<plugin>:hello` |
| Scope | one project | shareable via marketplace |
| Hooks | `settings.json` | `hooks/hooks.json` |
| Versioning | none | semver / git SHA |
| Distribution | manual copy | `/plugin install` |

### Agent frontmatter (official)
```yaml
---
name: agent-name
description: ...
model: sonnet
effort: medium
maxTurns: 20
disallowedTools: Write, Edit
isolation: worktree   # workspace isolation
---
```
Forbidden in plugin-shipped agents: `hooks`, `mcpServers`, `permissionMode`.

## 4. Comparative summary (cross-cut)

### Unit of composition
| Framework | Primary | Secondary |
|---|---|---|
| OMC | Agent lane (executor, planner, architect, …) | Skills invoke agents; hooks inject context |
| Superpowers | SKILL.md per engineering practice | Single SessionStart hook; no agents/, no MCP |
| SuperClaude | Slash command + persona | 20 agents; 8 MCP servers; Python installer |
| Official | Composable primitives (skills/agents/hooks/MCP/LSP/monitors) | None — no opinion |

### State handling
| Framework | State |
|---|---|
| OMC | Explicit state layer: `.omc/state/sessions/`, notepad, project-memory.json, SQLite for swarm/team |
| Superpowers | Stateless skills; uses built-in TodoWrite for in-session tracking only |
| SuperClaude | Behavioral injection via CLAUDE.md; persona-level, not session-state |
| Official | None built-in; `userConfig` for install-time config |

### Multi-agent orchestration
| Framework | Model |
|---|---|
| OMC | Named agent roster; team pipeline `plan→prd→exec→verify→fix(loop)`; `TeamCreate`/`SendMessage`; tmux multi-pane workers |
| Superpowers | Orchestration *inside* skills (e.g., `subagent-driven-development` is itself a multi-agent workflow doc); no central pipeline |
| SuperClaude | Auto-activated personas by keyword; no explicit pipeline |
| Official | Per-agent isolation (worktree); no pipeline; Claude picks agents from task context |

### Philosophy
- **Superpowers**: Skills as codified engineering culture. Process over prompting; behavioral evals decide content. Multi-harness markdown.
- **OMC**: Agent lanes as the unit of delegation. Verification before claiming completion. Heavy state infra. Explicit pipelines.
- **SuperClaude**: Personality transplant — commands + personas via meta-programming injection.
- **Official**: Neutral primitive set; the *community* owns the methodology.

### Where the orchestration lives — the key axis
- Superpowers: orchestration lives **inside skills** (one SKILL.md = one workflow).
- OMC: orchestration lives **outside skills**, in agent roles + team pipeline + state infrastructure; skills are entry points.

### All sources
- https://github.com/obra/superpowers
- https://github.com/obra/superpowers-marketplace
- https://github.com/obra/superpowers-skills
- https://blog.fsck.com/2025/10/09/superpowers/
- https://github.com/SuperClaude-Org/SuperClaude_Framework
- https://code.claude.com/docs/en/plugins
- https://code.claude.com/docs/en/plugins-reference
- https://github.com/anthropics/claude-plugins-official
- https://agentskills.io/specification
- https://github.com/hesreallyhim/awesome-claude-code (landscape index)
