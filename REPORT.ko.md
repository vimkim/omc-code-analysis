# 큰 Claude Code 플러그인은 어떻게 만들어지는가 — oh-my-claudecode 해부와 나만의 OMC를 만들기 위한 학습 로드맵

> Claude Code를 매일 쓰지만 큰 플러그인을 만들어 본 적 없는 분께 드리는 가이드입니다. OMC와 Superpowers, SuperClaude 코드를 직접 읽고 정리한 노트라서, 추측이 아니라 파일 경로와 라인을 따라 읽으시면 됩니다.

## 0. 한 페이지 요약

- 플러그인은 본 보고서가 본문에서 다루는 핵심 6개 확장 포인트(`skills/`, `commands/`, `agents/`, `hooks/hooks.json`, `.mcp.json`, `.lsp.json`) 위에서 동작합니다. 그 외 `bin/`, `monitors/monitors.json`(experimental), `themes/`(experimental)도 매니페스트가 인정하지만 본 보고서에서는 다루지 않습니다.
- OMC는 에이전트 레인을 1차 조립 단위로 두고, 그 위에 상태와 팀 인프라를 얹은 무거운 구조입니다.
- Superpowers는 SKILL.md 한 장이 워크플로 전체를 짊어지는 가벼운 구조이고, agents/와 MCP가 아예 없습니다.
- 이 둘을 가르는 한 축은 "오케스트레이션이 어디에 사는가"입니다. 스킬 *안*에 있으면 Superpowers, 스킬 *밖*(에이전트+팀+상태)에 있으면 OMC입니다.
- 학습 로드맵은 6단계입니다: Stage 1 최소 install → Stage 2 UserPromptSubmit 주입 → Stage 3 agents/+disallowedTools READ-ONLY → Stage 4 자체 MCP → Stage 5 PRD 루프 → Stage 6 미니 team(plan→exec→verify, bounded fix).

## 1. Claude Code 플러그인의 해부도

플러그인 구조의 본문 핵심 6개 확장 포인트는 다음과 같습니다. `skills/`(`/<plugin>:<skill>`로 부르는 SKILL.md 디렉터리), `commands/`(평면 .md 파일이며 점차 skills로 흡수되는 구식 슬롯), `agents/`(YAML 프론트매터를 단 마크다운), `hooks/hooks.json`(약 28종의 라이프사이클 이벤트 등록부), `.mcp.json`(외부 도구 서버 등록), `.lsp.json`(언어 서버 기반 코드 인텔)입니다. 그 외 매니페스트가 인정하는 슬롯으로는 `bin/`(플러그인 활성 동안 PATH에 추가되는 실행 파일 디렉터리), `outputStyles`(스타일 디렉터리 포인터), `experimental.monitors`(`monitors/monitors.json`), `experimental.themes`(`themes/`), 그리고 `userConfig`/`channels`/`dependencies` 같은 주변 필드가 있지만, 본 보고서는 이들을 본문에서 다루지 않습니다.

`plugin.json`에서 필수 필드는 kebab-case `name` 하나뿐입니다. 나머지는 모두 선택이지만 큰 플러그인은 보통 `version`, `description`, `author`, `homepage`, `repository`, `license`, `keywords`와 함께 확장 포인트 포인터 `skills`, `commands`, `agents`, `hooks`, `mcpServers`, `lspServers`, `outputStyles`, `experimental.{themes,monitors}`를 채웁니다. 한층 더 가면 설치 시 입력을 받는 `userConfig`(예: `sensitive: true`로 토큰을 가림), 플러그인 간 메시지 버스 `channels`, semver 의존성 `dependencies`도 있습니다. 런타임에는 `${CLAUDE_PLUGIN_ROOT}` 변수가 설치된 플러그인 경로로 치환된다는 점만 기억하시면 됩니다.

표준(`.claude/`) 설정과 플러그인 패키징의 차이는 세 줄로 요약됩니다. 호출은 `/hello` 대 `/<plugin>:hello`, 훅은 `settings.json` 대 `hooks/hooks.json`, 배포는 수동 복사 대 `/plugin install`입니다.

출처: code.claude.com/docs/en/{plugins,plugins-reference}; github.com/anthropics/claude-plugins-official; https://agentskills.io/specification

## 2. oh-my-claudecode 깊이 읽기

### 2.1 패키징

```json
{ "name": "oh-my-claudecode", "version": "4.13.2", "skills": "./skills/", "mcpServers": "./.mcp.json" }
```
출처: `.claude-plugin/plugin.json`

OMC는 플러그인이면서 동시에 npm 패키지로도 배포됩니다. `bin`에 `omc`, `omc-cli`, `oh-my-claudecode`가 모두 `bridge/cli.cjs`로 연결되어 있어, 플러그인 외부에서도 같은 도구를 부를 수 있습니다. 빌드 체인은 `tsc → build-skill-bridge → build-mcp-server → build-bridge-entry → compose-docs → build-runtime-cli → build-team-server → build-cli` 순서로 흐릅니다.
출처: `package.json` scripts

### 2.2 Skills 레이어

```yaml
---
name: ralph
description: Self-referential loop until task completion with configurable verification reviewer
level: 4
---
```
출처: `skills/ralph/SKILL.md`

OMC는 `level` 필드를 자체 컨벤션으로 둡니다. 1은 가벼운 도우미, 4는 PRD 루프 같은 무거운 워크플로입니다. `ralph`(L4)는 `.omc/prd.json`을 만들어 스토리 단위로 반복하며, Step 7 → 7.5 → 7.6 → 8을 한 턴에 이어 붙이고 "boulder never stops" 신호로 조기 종료를 막습니다. `autopilot`(L4)은 6단계(Expansion → Planning → Execution → QA → Validation → Cleanup)로 흐릅니다. Phase 0은 ralplan consensus plan 또는 deep-interview spec이 이미 있으면 analyst와 architect 단계를 건너뜁니다. Phase 4는 architect, security-reviewer, code-reviewer를 병렬로 호출해 세 명 모두 승인해야 통과합니다. `cancel`(L2)은 `state_list_active`로 활성 모드를 읽어 의존성 역순으로 풀어내립니다.

키워드 라우팅은 두 훅이 분담합니다. `keyword-detector.mjs`(UserPromptSubmit, 5초)가 우선순위 키워드(cancelomc/ralph/autopilot/ultrawork/ulw/ccg/ralplan/deep interview/deslop)를 잡아 `.omc/state/sessions/{sessionId}/skill-active-state.json`에 기록하고, `skill-injector.mjs`(UserPromptSubmit, 3초)가 `~/.claude/skills/omc-learned/`, `~/.omc/skills/`, `.omc/skills/`를 재귀 스캔해 일치하는 SKILL.md를 `<system-reminder>`로 주입합니다.
출처: `scripts/{keyword-detector,skill-injector}.mjs`

### 2.3 Agents 레이어

```yaml
---
name: architect
model: opus
level: 3
disallowedTools: Write, Edit
---
```
출처: `agents/architect.md`

핵심은 "도구 권한으로 역할을 강제한다"입니다. 프롬프트로만 "쓰지 마세요"라고 부탁하는 게 아니라 `disallowedTools: Write, Edit`로 실제로 막아 둡니다. `architect`는 그래서 architect/debug 자문 역할로 정의되고, 같은 문제로 3회 실패하면 회로를 차단해 다른 레인으로 넘깁니다. `executor`(sonnet, level 2)는 "최소 변경, 혼자 일함, 인접 코드 리팩토링 금지, 같은 문제 3회 실패 시 architect로 에스컬레이션"을 명시한 작업자입니다. 오케스트레이터는 `src/agents/preamble.ts`의 `wrapWithPreamble()`로 작업자 프롬프트를 감싸 재귀 sub-agent 생성을 차단합니다. 공식 스키마는 여기에 더해 `effort`, `maxTurns`, `isolation`(예: `worktree`) 같은 선택 필드도 받습니다.

레인은 docs/ARCHITECTURE.md 기준으로 Build/Analysis(explore, analyst, planner, architect, debugger, executor, verifier, tracer), Review(security-reviewer, code-reviewer), Domain(test-engineer, designer, writer, qa-tester, scientist, git-master, document-specialist, code-simplifier), Coordination(critic)으로 나뉩니다.
출처: `agents/{architect,executor}.md`, `docs/ARCHITECTURE.md`

### 2.4 Hooks 파이프라인

| 이벤트 | 스크립트(타임아웃) |
| --- | --- |
| UserPromptSubmit | keyword-detector.mjs(5s), skill-injector.mjs(3s) |
| SessionStart | session-start.mjs, project-memory-session.mjs, wiki-session-start.mjs |
| PreToolUse / PermissionRequest | pre-tool-enforcer.mjs(3s); permission-handler.mjs |
| PostToolUse | post-tool-verifier, project-memory-posttool, post-tool-rules-injector |
| SubagentStart/Stop, Stop | subagent-tracker, verify-deliverables, context-guard-stop, persistent-mode, code-simplifier |
| PreCompact / SessionEnd | pre-compact, project-memory-precompact, wiki-pre-compact; session-end(30s), wiki-session-end(30s) |

출처: `hooks/hooks.json`

훅은 JSON으로 `<system-reminder>`를 뱉어 대화 중간에 주입됩니다. 자주 보이는 신호는 `[MAGIC KEYWORD: autopilot]`, `The boulder never stops`, 그리고 메모리용 `<remember>`(7일 TTL)와 `<remember priority>`(영구)입니다. 비상시에는 환경변수 `DISABLE_OMC=1`로 전체를 끄거나 `OMC_SKIP_HOOKS=keyword-detector,skill-injector`처럼 콤마로 일부만 끌 수 있습니다.

워크트리 안전장치는 `post-tool-rules-injector`가 담당합니다. 이 훅은 `data.cwd` 대신 접근된 파일 경로에서 위로 거슬러 올라가며, 워크트리 루트의 `.git` *파일*(디렉터리가 아님)을 만나면 멈춥니다. 부모 리포의 규칙이 워크트리로 새는 것을 막기 위함입니다.

### 2.5 MCP bridge

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
출처: `.mcp.json`

서버 별칭이 `t`라서, 도구는 `mcp__plugin_oh-my-claudecode_t__*` 형태로 노출됩니다. `bridge/`는 esbuild로 번들된 .cjs 파일들이 모인 디렉터리이고, 각 파일이 한 가지 책임을 집니다. `bridge/mcp-server.cjs`(963KB)는 state_*, notepad_*, project_memory_*, wiki_*, lsp_*, ast_grep_*, python_repl을 한데 묶은 메인 번들입니다. `bridge/team-mcp.cjs`(657KB)는 TeamCreate, SendMessage, TaskCreate/List/Get/Update와 inbox/outbox, 하트비트를 담당합니다. `bridge/cli.cjs`(3.14MB)는 `omc` CLI 본체이고, `bridge/runtime-cli.cjs`(271KB)는 tmux 팀 워커가 부팅될 때 쓰는 진입점, `bridge/team-bridge.cjs`(81KB)는 팀 브리지 데몬의 라이프사이클을 관리합니다.
출처: `.mcp.json`, `bridge/*.cjs`

### 2.6 src/ TypeScript 레이아웃

`src/`는 약 20개 디렉터리로 나뉘어 있고, 한 번에 다 읽기보다는 다음 순서로 따라 읽으시기를 추천드립니다.

1. `src/mcp/omc-tools-server.ts` — 도구 하나를 등록하는 가장 작은 살아 있는 예시입니다. 직접 MCP를 만들 때 형판 삼기 좋습니다.
2. `src/hooks/keyword-detector/` — stdin JSON을 읽고 우선순위 매칭으로 `<system-reminder>`를 뱉는 패턴이 잘 보입니다.
3. `src/hooks/skill-injector/` — 재귀적으로 SKILL.md 프론트매터를 스캔하고 중복을 거르는 절차가 들어 있습니다.
4. `src/team/runtime.ts`와 `src/team/tmux-session.ts` — 멀티 페인 워커 오케스트레이션의 골격이 여기서 시작됩니다.

이 네 곳만 읽어도 "키워드 → 상태 → 스킬 주입 → 팀 워커"라는 OMC 흐름의 80%가 손에 잡힙니다. 그 외 디렉터리(features/, verification/, planning/, providers/ 등)는 필요할 때 따라가시면 됩니다.
출처: `src/{mcp,hooks/keyword-detector,hooks/skill-injector,team}` 트리

### 2.7 .omc/ 런타임 아티팩트와 team 파이프라인

```
.omc/
├── state/sessions/{sessionId}/{autopilot|ralph|ultrawork|ultraqa|team|ralplan|skill-active|boulder|cancel-signal}-state.json
├── state/swarm.db
├── state/team-bridge/{team}/*.heartbeat.json
├── notepad.md
├── project-memory.json
├── plans/, research/, logs/, specs/, handoffs/, skills/
```
출처: `.omc/research/plugin-frameworks-internal-omc.md` §7

세션 단위로 모드 상태가 분리되어 있어 한 머신에서 여러 모드를 동시에 돌려도 충돌하지 않습니다. Team 파이프라인은 `team-plan → team-prd → team-exec → team-verify → team-fix(bounded loop)`로 흐르며, 단계별 산출물은 `.omc/handoffs/{team-plan,team-prd,team-exec}.md`에 떨어집니다.

### 2.8 빌드/릴리스 파이프라인

`build-*.mjs`는 모두 esbuild 호출이고, 각 스크립트가 `bridge/*.cjs` 번들 하나씩을 만듭니다. `compose-docs.mjs`는 `{{INCLUDE:partials/...}}` 템플릿 보간으로 `docs/templates/*.template.md`를 `docs/*.md`로 합성하고, 같은 partial을 `docs/shared/`에도 복사해 스킬에서 재사용할 수 있게 만듭니다. `sync-version.sh`는 `package.json`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` 세 매니페스트의 버전을 한 번에 맞춰 주고, `npm version`이 호출합니다. `release.ts`는 bump → build → docs → tests → tag → publish 순서로 흐르는 한 줄짜리 릴리스 컨베이어입니다.
출처: `scripts/{compose-docs,sync-version,release}.{mjs,sh,ts}`, `package.json` scripts

### 2.9 훔쳐올 만한 행동 패턴 7개

1. Worker Preamble Protocol — 오케스트레이터가 작업자 프롬프트를 `wrapWithPreamble()`로 감싸 재귀 sub-agent 생성을 봉합니다.
2. Delegation Enforcer — 에이전트 프롬프트에 "당신이 책임지지 *않는* 일" 절을 명시하고, `disallowedTools`로 한 번 더 잠급니다.
3. Magic-keyword routing — UserPromptSubmit에서 우선순위 매칭으로 키워드를 잡아 상태 파일에 기록하고, 다음 훅이 그 상태를 보고 컨텍스트를 채웁니다.
4. System-reminder 주입 — 훅이 표준출력으로 JSON을 흘리면 Claude Code가 대화 중간에 시스템 리마인더로 끼워 넣습니다. CLAUDE.md를 건드리지 않고도 런타임 가이드를 줄 수 있습니다.
5. `<remember>` 태그 — 짧게 7일 TTL, `<remember priority>`는 영구 저장입니다.
6. 두 단계 킬 스위치 — 전체를 끄는 `DISABLE_OMC`와, 특정 훅만 끄는 `OMC_SKIP_HOOKS`(콤마 구분)로 비상 정지를 분리해 둡니다.
7. PRD-boulder 루프 — 스토리 단위 반복과 "boulder never stops" 리마인더를 묶어 검증 통과 전까지 멈추지 못하게 합니다.

보너스로, 워크트리 안전한 프로젝트 루트(`.git` 파일 마커), PreCompact 요약과 notepad 자동 prune, 모드 재수화, autopilot Phase 3의 bounded fix(최대 5회 사이클, 같은 에러가 3회 반복되면 fundamental-issue 보고서로 탈출), Phase 4의 병렬 합의 게이트도 비슷한 결의 패턴입니다.
출처: `skills/autopilot/SKILL.md`, `src/agents/preamble.ts`, `hooks/hooks.json`

## 3. Superpowers 깊이 읽기

레포는 https://github.com/obra/superpowers, 디자인 글은 https://blog.fsck.com/2025/10/09/superpowers/ 입니다.

레포 트리는 `.claude-plugin/{plugin,marketplace}.json` + `skills/`(14개: brainstorming, subagent-driven-development, test-driven-development, writing-plans, writing-skills, dispatching-parallel-agents, executing-plans, finishing-a-development-branch, receiving-code-review, requesting-code-review, systematic-debugging, using-git-worktrees, using-superpowers, verification-before-completion) + `hooks/{hooks.json, session-start}` + `CLAUDE.md`/`AGENTS.md` + 다중 하네스 매니페스트(`.codex-plugin/`, `.cursor-plugin/`, `.opencode/`, `gemini-extension.json`)로 구성됩니다. **`agents/`, `commands/`, MCP, `src/`, 빌드 파이프라인이 전부 없습니다.** 가벼움이 곧 정체성입니다.

```json
{ "name": "superpowers", "version": "5.1.0",
  "description": "Core skills library for Claude Code: TDD, debugging, collaboration patterns, and proven techniques",
  "author": { "name": "Jesse Vincent", "email": "jesse@fsck.com" },
  "license": "MIT",
  "keywords": ["skills", "tdd", "debugging", "collaboration", "best-practices", "workflows"] }
```
출처: https://github.com/obra/superpowers/blob/main/.claude-plugin/plugin.json

```json
{ "hooks": { "SessionStart": [{ "matcher": "startup|clear|compact",
  "hooks": [{ "type": "command",
    "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start" }] }] } }
```
출처: https://github.com/obra/superpowers/blob/main/hooks/hooks.json

PreToolUse도 PostToolUse도 UserPromptSubmit도 없습니다. 훅 이벤트 28종 중 1종(SessionStart)만 사용하고, 행동의 무게는 전부 스킬 본문이 짊어집니다.

```yaml
---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---
```
출처: https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md

`writing-skills/SKILL.md`에는 흥미로운 회고가 있습니다. description이 워크플로 자체를 요약해 버리면 모델이 description만 보고 본문을 안 읽었다는 것입니다. 짧고 좁게 "언제 쓰는가"만 적어 두자 모델이 본문 flowchart를 제대로 따라 읽었습니다. 그래서 결론이 "Writing skills IS Test-Driven Development applied to process documentation."입니다. 즉 **스킬 description은 행동 게이트**이고, 좁게 적어야 본문이 읽힙니다.
출처: https://github.com/obra/superpowers/blob/main/skills/writing-skills/SKILL.md

Jesse Vincent의 디자인 글 핵심도 같은 결입니다. "Skills are what give your agents Superpowers." 자기 강화 루프(에이전트가 새 스킬을 같이 쓰며 다음 스킬을 더 싸게 쓰게 됨), "책 한 권을 모델에 건네며 '읽고 새로 배운 걸 적어 두라'고 시킬 수 있다"는 비유, 그리고 개인용 스킬은 `~/.claude/skills`, 배포용은 플러그인의 `skills/`라는 분리가 그 글의 뼈대입니다. CLAUDE.md에는 한 발 더 들어가서 "Anthropic 공식 스타일에 맞추려는 PR은 behavioral eval 증거 없이 받지 않는다"고 못 박혀 있습니다. **스킬은 '보기 좋은 문서'가 아니라 행동 평가로 튜닝된 프롬프트**라는 선언입니다.
출처: https://blog.fsck.com/2025/10/09/superpowers/, https://github.com/obra/superpowers/blob/main/CLAUDE.md

마지막으로 다중 하네스: 같은 SKILL.md를 Codex CLI, Codex App, Gemini CLI, OpenCode, Cursor, GitHub Copilot CLI 매니페스트에 동시에 얹어 배포합니다. 마크다운 한 장이 여러 하네스에 살게 됩니다.

## 4. 비교: OMC vs Superpowers vs SuperClaude vs 공식

조립 단위:

| | 1차 | 2차 |
| --- | --- | --- |
| OMC | 에이전트 레인(executor/planner/architect 등) | 스킬이 에이전트를 호출, 훅이 컨텍스트 주입 |
| Superpowers | 엔지니어링 관행마다 SKILL.md 한 장 | SessionStart 단일 훅, agents/와 MCP 없음 |
| SuperClaude | 슬래시 명령(`/sc:*`) + 페르소나 | 20 에이전트, 8 MCP, Python 인스톨러 |
| Official | 합성 가능 프리미티브(skills/agents/hooks/MCP/LSP/monitors) | 공식 의견 없음 |

상태:

| | 상태 |
| --- | --- |
| OMC | 명시적 상태 레이어(`.omc/state/sessions/`, notepad, project-memory.json, swarm/team용 SQLite) |
| Superpowers | 무상태, 세션 내 추적은 내장 TodoWrite만 사용 |
| SuperClaude | CLAUDE.md 행동 주입, 페르소나 단위이고 세션 상태 아님 |
| Official | 내장 없음, 설치 시 입력만 `userConfig` |

오케스트레이션:

| | 모델 |
| --- | --- |
| OMC | 명명된 에이전트 명단 + `team-plan→prd→exec→verify→fix(bounded loop)` + TeamCreate/SendMessage + tmux 멀티 페인 |
| Superpowers | 오케스트레이션이 *스킬 안*(예: `subagent-driven-development`가 그 자체로 멀티에이전트 워크플로 문서) |
| SuperClaude | 키워드로 페르소나 자동 활성, 명시적 파이프라인 없음 |
| Official | 에이전트별 격리(`isolation: worktree`), 파이프라인 없음, 작업 컨텍스트로 에이전트 선택 |

철학:

| | 한 줄 |
| --- | --- |
| OMC | 에이전트 레인을 위임 단위로, 검증 게이트를 통과 조건으로, 무거운 상태 인프라를 기억으로 둔다 |
| Superpowers | 스킬을 행동 평가로 튜닝되는 "엔지니어링 문화의 코드화" 단위로, process over prompting으로 운영하고 다중 하네스로 재배포한다 |
| SuperClaude | behavioral instruction injection과 component orchestration으로 "Claude Code를 구조화된 개발 플랫폼으로 변형"한다 |
| Official | 중립 프리미티브 세트만 제공하고 방법론은 커뮤니티에 맡긴다 |

**핵심 축은 "어디에 오케스트레이션이 사는가"입니다.** Superpowers는 *스킬 안*, OMC는 *스킬 밖*(에이전트 + 팀 + `.omc/state/` + MCP + 훅)에 둡니다. 이 한 축이 무게, 자유도, 학습 비용을 거의 다 설명합니다.

## 5. 있는 것과 없는 것의 차이

OMC만 가진 것은 다섯 갈래입니다.

- 키워드 라우팅: UserPromptSubmit에서 우선순위 매칭으로 모드 진입을 자동화합니다.
- 상태 보존: state-sessions와 PreCompact 요약이 결합해 컨텍스트 압축 후에도 모드를 재수화합니다.
- 검증 게이트: autopilot Phase 4의 architect+security-reviewer+code-reviewer 병렬 합의로 "자기 승인" 함정을 차단합니다.
- 팀 파이프라인: plan→prd→exec→verify→fix 흐름에 TeamCreate/SendMessage/TaskCreate, tmux 멀티 페인, 하트비트가 묶여 있습니다.
- 메모리 MCP + 두 단계 킬 스위치: notepad/project-memory/wiki를 도구로 노출하고, `DISABLE_OMC`와 `OMC_SKIP_HOOKS`로 비상 정지를 분리합니다.

Superpowers만 가진 것은 다음 세 가지가 두드러집니다.

- 다중 하네스: Codex/Gemini/OpenCode/Cursor/Copilot CLI에 매니페스트만 바꿔 같은 SKILL.md를 재배포할 수 있어, 마크다운 한 장이 여러 도구에 동시에 산다는 점이 가장 강력합니다.
- 가벼움: agents/, commands/, MCP, src/가 모두 없으니 학습 곡선이 사실상 "SKILL.md를 잘 쓰는 법" 한 가지로 좁혀집니다.
- 행동 평가 튜닝: PR을 behavioral eval 증거로만 받기 때문에, 스킬 본문이 시간이 갈수록 "느낌"이 아니라 측정값에 따라 깎입니다.

한 줄 요약은 "**OMC는 인프라, Superpowers는 콘텐츠**"입니다.

## 6. 나만의 OMC 만들기 — 학습 로드맵 6단계

### Stage 1 — 최소 install

**산출물:** `my-omc/.claude-plugin/{plugin.json, marketplace.json}`과 `my-omc/skills/hello/SKILL.md` 한 장.
**성공 기준:** `/plugin marketplace add ./my-omc/.claude-plugin/marketplace.json` 후 `/plugin install`이 통과하고, `/<plugin>:hello`로 스킬이 호출됩니다.
**흔한 실수:**
- `name`이 kebab-case가 아니면 설치가 침묵 실패합니다 — 설치 후 `/plugin list`에 안 보이면 이걸 의심하세요.
- description에 워크플로 전체를 적어 버리면 모델이 본문을 안 읽습니다 — "언제 쓰는가"만 한 줄로 좁히세요.
- marketplace.json의 `source: "./"` 오타로 플러그인을 못 찾는 경우가 잦습니다 — 설치 에러에 "plugin not found"가 보이면 여기를 보세요.
**다음 단계 트리거:** "키워드만 쳐도 자동으로 떠야겠다"는 욕구가 생기면 Stage 2로 넘어갑니다.

### Stage 2 — UserPromptSubmit 주입

**산출물:** `+hooks/hooks.json`과 `+hooks/inject-on-keyword.cjs`.
**성공 기준:** 사용자가 미리 정한 키워드를 치면 훅이 `<system-reminder>`를 표준출력으로 흘려 다음 턴에 주입됩니다.
**흔한 실수:**
- 타임아웃을 안 정해 두면 훅이 5초 이상 걸려 Claude Code가 죽은 줄 압니다 — 첫 버전은 5s/3s 정도로 보수적으로 잡으세요.
- 훅 출력이 유효 JSON이 아니면 조용히 버려집니다 — `node hook.cjs <<<'{}'` 식으로 단독 실행해 stdout이 깨끗한지 확인하세요.
- `.claude/settings.json`에만 등록하고 `hooks/hooks.json`을 빼먹으면 플러그인 설치 시 안 따라옵니다 — 둘이 동기화되었는지 보세요.
**다음 단계 트리거:** "역할별로 일꾼을 따로 두고 싶다"는 욕구가 생기면 Stage 3로 갑니다.

### Stage 3 — agents/ + disallowedTools READ-ONLY

**산출물:** `+agents/reader.md` 같은 첫 에이전트 한 명.
**성공 기준:** 프론트매터에 `disallowedTools: Write, Edit`을 두고, `<Role>`/`<Constraints>`/`<Output_Format>` 섹션과 "3회 실패 시 에스컬레이션" 규칙을 본문에 명시합니다.
**흔한 실수:**
- `disallowedTools`를 콤마 없이 한 줄로 적으면 파서가 토큰 하나로 봅니다 — 정확히 `Write, Edit`처럼 콤마+공백을 두세요.
- 본문에 "모든 일을 다 한다"고 적어 두면 작업이 비대해집니다 — 한 가지 책임만 적고 나머지는 "책임지지 않는 일"에 옮기세요.
- 플러그인에 동봉되는 에이전트는 `hooks`, `mcpServers`, `permissionMode`가 금지입니다 — 무심코 넣으면 설치 시 거부됩니다.
**다음 단계 트리거:** "세션 간 메모를 다시 읽고 싶다"는 욕구가 생기면 Stage 4로 갑니다.

### Stage 4 — 자체 MCP

**산출물:** `+.mcp.json`과 `+bridge/mcp-server.cjs`(처음에는 작은 stdio 서버 한 개).
**성공 기준:** `state_read`/`state_write` 두 도구를 등록해 세션별 JSON 파일에 읽고 씁니다. `src/mcp/omc-tools-server.ts`를 형판으로 두고 따라 쓰시면 됩니다. 도구는 `mcp__plugin_<your>_t__*`로 노출됩니다.
**흔한 실수:**
- 도구의 입력 스키마(JSON Schema) 선언을 빼먹으면 호출 측이 검증 에러로 떨어집니다 — 모든 도구는 `inputSchema`를 채우세요.
- JSON 직렬화가 안 되는 값(함수, 순환 객체)을 그대로 반환하면 stdio가 깨집니다 — 항상 plain object로 바꿔 반환하세요.
- 디버깅용 `console.log`를 stdout으로 그대로 두면 MCP stdio 프레임이 망가집니다 — 로그는 stderr나 파일로 보내세요.
- 세션 ID를 전역 변수에 캐시해 두면 다중 세션에서 섞입니다 — 인자로 받은 sessionId만 신뢰하세요.
**다음 단계 트리거:** "PRD를 외재화해 반복하고 싶다"는 욕구가 생기면 Stage 5로 갑니다.

### Stage 5 — PRD 루프

**산출물:** `+skills/mini-ralph/SKILL.md`. 시작 파일로 `.omc/prd.json`을 스캐폴딩하시는 걸 추천드립니다. 스키마는 최소한 `stories[]`(각 항목 `id/title/acceptanceCriteria`), `passes`(반복 횟수), 전체 `acceptanceCriteria`만 두면 충분합니다. 본문은 OMC의 `skills/ralph/SKILL.md`를 그대로 거울처럼 두고 단계 이름과 신호 문구를 그대로 베끼세요.
**성공 기준:** 7단계(PRD 작성 → 수용 기준 정련 → 스토리 분해 → 실행 → 검증 → 반영 → 다음 스토리)가 한 턴 안에서 이어 붙고, 매 턴에 "The boulder never stops" 리마인더가 주입되며, 단계 진행이 `state_write`로 세션 상태에 기록됩니다.
**흔한 실수:**
- description에 7단계 흐름을 다 적어 버리면 모델이 본문 flowchart를 안 읽습니다 — "언제 쓰는가"로만 좁히세요.
- description은 짧고 좁게, 본문은 한 워크플로 단위로 끊어 쓰세요. 너무 길면 라우터가 적절한 스킬을 고르기 어려워집니다.
- "boulder never stops" 리마인더 없이 정중한 종료 메시지를 허용하면 검증 전에 멈춥니다. 게다가 reviewer-verified 완료 게이트가 없으면 모델이 "할 만큼 했다"에서 끝냅니다 — 두 신호를 같이 묶어 두세요.
**다음 단계 트리거:** "단계별 에이전트로 분업하고 싶다"는 욕구가 생기면 Stage 6로 갑니다.

### Stage 6 — 미니 team(plan→exec→verify, bounded fix)

**산출물:** `+skills/mini-team/SKILL.md`, `+agents/{planner,executor,verifier}.md`, `+handoffs/{plan,exec,verify}.md`.
**성공 기준:** 스킬이 세 에이전트를 순서대로 호출하고 각 단계 산출물을 `handoffs/`에 떨어뜨립니다. fix는 N회로 상한이 걸리고, 같은 에러가 3회 반복되면 fundamental-issue 보고서로 빠져나옵니다. planner는 `disallowedTools: Write, Edit`로 READ-ONLY를 강제합니다.
**흔한 실수:**
- fix 루프 상한이 없으면 무한 반복으로 토큰을 태웁니다 — `maxAttempts`를 코드와 본문 양쪽에 적으세요.
- 단계마다 같은 파일을 덮어쓰면 디버깅이 불가합니다 — 단계별로 파일명을 분리하세요(plan.md / exec.md / verify.md).
- planner에 Write/Edit이 열려 있으면 계획 단계에서 코드를 만지기 시작합니다 — 프론트매터로 한 번 더 잠그세요.
**다음 단계 트리거:** 같은 SKILL.md를 Codex/Gemini 매니페스트에 얹어 다중 하네스 모방을 시도하거나, description A/B로 라우팅 정확도를 측정해 보세요.

## 7. 다음 2주 학습 계획

### 7.0 이번 주(휴일) — 총 1.5시간

휴일이라 코드는 거의 안 봅니다. 그 대신 "플러그인이라는 게 뭔지"를 머릿속에 큰 그림으로 그리는 시간만 가집시다.

- 화 30분: https://blog.fsck.com/2025/10/09/superpowers/ 정독.
- 목 30분: code.claude.com/docs/en/plugins-reference의 manifest 스키마와 hook event 28종 표 훑어보기.
- 토 30분: 본 보고서 §1과 §4 다시 읽기.

### 7.1 다음 주: Superpowers/SuperClaude 읽기 — 총 9시간

- 월 2h: Superpowers, SuperClaude, OMC, claude-plugins-official, agentskills.io 다섯 곳의 `plugin.json`을 한 페이지로 비교 정리.
- 화 1.5h: Superpowers의 `skills/{subagent-driven-development, writing-skills, verification-before-completion}/SKILL.md` 정독, 메모는 자신의 `~/.claude/skills`로.
- 수 1h: Superpowers `hooks/hooks.json`(SessionStart 단일)과 OMC `hooks/hooks.json`(다단계) 차이 표로 비교.
- 목 2h: SuperClaude_Framework의 README "30 commands / 20 agents / 7 modes / 8 MCP" 카운트를 OMC 카탈로그와 매핑하고, **`SuperClaude_Framework/src/`(Python 인스톨러)와 `plugins/` 또는 `skills/` 중 임의의 한 페르소나/명령 파일을 1개 골라 직접 열어 읽고 100줄 요약**합니다. 파일 경로와 함수 이름을 메모에 남기세요.
- 금 1.5h: 본 보고서 §4의 4개 표(조립/상태/오케스트레이션/철학)를 빈 표로 만들고 손으로 다시 채워 비교.
- 토 1h: 주간 회고 메모, 모르는 단어 5개 정리.
- 일: 휴식.
- **주말 게이트:** Superpowers와 SuperClaude의 "조립 단위" 차이를 한 문단으로 적을 수 있고, SuperClaude `src/`에서 본 파일 이름을 셋 이상 댈 수 있으면 다음 주로 갑니다.

### 7.2 그 다음 주: 훅 만지기 + Stage 1~2 — 총 10시간

- 월 2h: Stage 1(최소 install, hello 스킬) 끝까지.
- 화 2h: SessionStart 훅 한 줄짜리 셸 스크립트로 부팅 메시지 인쇄.
- 수 2h: UserPromptSubmit 훅 추가, 5초 안에 끝나는지 측정.
- 목 1h: OMC `scripts/keyword-detector.mjs`를 읽고 자신의 키워드 매처로 한 번 따라 적기.
- 금 2h: 두 훅을 합쳐 미니 키워드 라우팅 완성.
- 토 1h: 주간 회고 + 다음 주 Stage 3 계획.
- **주말 게이트:** Stage 1과 Stage 2의 hello 스킬이 `/plugin install` 후 키워드 트리거에 응답하면 Stage 3로 이동합니다.

## 8. 부록: 인용한 모든 파일/URL

**내부**: `.omc/research/plugin-frameworks-{internal-omc,external}.md`, `.claude-plugin/{plugin,marketplace}.json`, `.mcp.json`, `package.json`, `hooks/hooks.json`, `agents/{executor,architect}.md`, `skills/{ralph,autopilot,cancel}/SKILL.md`, `scripts/{keyword-detector,skill-injector,post-tool-rules-injector,compose-docs,sync-version,release}.{mjs,sh,ts}`, `src/{agents/preamble,mcp/omc-tools-server,hooks/{keyword-detector,skill-injector},team/{runtime,tmux-session}}.ts`, `bridge/*.cjs`, `docs/ARCHITECTURE.md`, `.omc/`.

**외부**: https://github.com/obra/superpowers, https://github.com/obra/superpowers-marketplace, https://github.com/obra/superpowers-skills, https://blog.fsck.com/2025/10/09/superpowers/, https://github.com/obra/superpowers/blob/main/.claude-plugin/plugin.json, https://github.com/obra/superpowers/blob/main/hooks/hooks.json, https://github.com/obra/superpowers/blob/main/CLAUDE.md, https://github.com/obra/superpowers/blob/main/skills/{brainstorming,subagent-driven-development,writing-skills}/SKILL.md, https://github.com/SuperClaude-Org/SuperClaude_Framework, https://code.claude.com/docs/en/plugins, https://code.claude.com/docs/en/plugins-reference, https://github.com/anthropics/claude-plugins-official, https://github.com/hesreallyhim/awesome-claude-code.

**스펙 참고 한 줄**: https://agentskills.io/specification — 스킬 프론트매터 공식 스펙(본 보고서 본문에서는 직접 인용하지 않으며, Superpowers의 `writing-skills/SKILL.md`가 참조하는 공식 스펙입니다).
