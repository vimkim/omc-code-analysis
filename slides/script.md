# 발표 스크립트 — 큰 Claude Code 플러그인은 어떻게 만들어지는가

발표자 메모: 한국어, 존댓말, 친근한 동료 톤. 슬라이드당 50~60초, 100~150자 내외.

## S1. 타이틀

안녕하세요, 김대현입니다. 오늘 15분 동안 "큰 Claude Code 플러그인은 어떻게 만들어지는가"를 같이 보려고 합니다. 차주에 우리가 Superpowers랑 SuperClaude 코드를 직접 읽기로 했잖아요. 그 전에 큰 플러그인의 공통 구조를 한 번 정리하고 싶어서, 며칠 동안 OMC 코드를 따라 읽고 정리한 내용을 공유드립니다. 추측이 아니라 파일 경로 그대로 따라가시면 됩니다. 다음 슬라이드에서 다섯 줄짜리 결론부터 깔고 시작하겠습니다.

## S2. 한 줄 요약 (TL;DR)

먼저 결론부터 다섯 줄로 깔고 시작하겠습니다. 첫째, 플러그인 본문 핵심은 6개 슬롯입니다. 둘째, OMC는 에이전트 레인을 1차 조립 단위로 둔 무거운 구조이고, 셋째, Superpowers는 SKILL.md 한 장이 다 짊어지는 가벼운 구조입니다. 넷째, 둘을 가르는 한 축은 "오케스트레이션이 어디에 사는가"입니다. 마지막으로, 우리도 따라 만들 수 있는 6단계 로드맵이 있습니다. 이 다섯 줄을 머리에 넣고 다음 슬라이드의 해부도로 넘어가겠습니다.

## S3. Claude Code 플러그인 해부도

플러그인을 열어 보면 결국 6개 슬롯이 본문 핵심입니다. 다이어그램에서 보시듯이 `skills/`, `commands/`, `agents/`, `hooks/hooks.json`, `.mcp.json`, `.lsp.json`이 본문이고요. 그 외에 `bin/`은 안정 슬롯이고, `monitors/`랑 `themes/`는 experimental입니다. 출처는 code.claude.com 공식 reference예요. 우리가 큰 플러그인을 봤을 때 "복잡하다"고 느끼는 건 이 6개 슬롯이 한꺼번에 켜져 있기 때문이지, 슬롯 자체가 많은 건 아닙니다. 이제 매니페스트 두 장을 보시죠.

## S4. plugin.json / marketplace.json

매니페스트는 의외로 간단합니다. 필수 필드는 kebab-case `name` 하나뿐이고, 나머지는 다 선택입니다. 런타임에는 `${CLAUDE_PLUGIN_ROOT}`라는 변수가 설치된 플러그인 경로로 치환된다는 것만 기억하시면 됩니다. 호출 규칙도 간단해요. 표준 `.claude/`는 `/hello`로 부르는데, 플러그인은 `/<plugin>:<skill>` 형태로 namespacing이 붙습니다. 그리고 `marketplace.json`은 자체 호스팅 마켓플레이스 메타데이터인데요, OMC는 `plugins[]`에 `source: "./"`로 자기 자신을 한 번 등록하고 `category`를 productivity로 잡아 두었습니다. `/plugin marketplace add`가 이걸 읽어요. 다음은 OMC 안으로 들어갑니다.

## S5. OMC 깊이 보기 (1) 패키징 + Skills

이제 OMC 안으로 들어가 봅시다. 첫 번째 코드블록의 OMC `plugin.json`을 보시면, 이름은 oh-my-claudecode, 버전 4.13.2이고 `skills`랑 `mcpServers` 두 포인터가 박혀 있어요. 그 아래 트리 다이어그램이 OMC 레포 구조입니다. OMC는 플러그인이면서 동시에 npm 패키지로도 배포돼서, `bin`에 `omc`, `omc-cli`, `oh-my-claudecode`가 모두 같은 `bridge/cli.cjs`로 연결돼 있어요. `files` 화이트리스트에는 `dist, agents, bridge, commands, hooks, scripts, skills, templates, docs, .claude-plugin, .mcp.json, README.md, LICENSE`까지 들어가 있습니다. Skills 쪽에 재미있는 게 OMC만의 컨벤션 `level` 필드인데요, 1은 가벼운 도우미, 4는 PRD 루프 같은 무거운 워크플로입니다. 다음은 에이전트 레이어로 넘어갑니다.

## S6. OMC 깊이 보기 (2) Agents

OMC가 가장 자랑스러워 하는 부분이 에이전트 레이어입니다. 핵심은 "도구 권한으로 역할을 강제한다"예요. 그냥 프롬프트로 "쓰지 마세요" 하고 부탁하는 게 아니라, `disallowedTools: Write, Edit` 한 줄로 실제로 막아 둡니다. 그래서 architect는 진짜로 자문 역할만 하게 돼요. 그래프 다이어그램을 보시면 사용자 작업 하나가 explore → analyst → architect → executor → verifier로 흐르고, verifier가 실패하면 executor로 되돌아가는 게 한 눈에 보이실 겁니다. 레인은 4개로 나뉘어 있는데, Build/Analysis가 우리가 가장 자주 만나는 일꾼들이고, Review는 게이트, Domain은 전문가, Coordination은 한 명짜리 critic입니다. 이제 OMC의 진짜 비밀 무기, 훅으로 넘어갑니다.

## S7. OMC 깊이 보기 (3) Hooks

여기서 잠시 가장 중요한 부분을 짚고 갈게요. Hooks가 OMC의 진짜 비밀 무기입니다. 시퀀스 다이어그램을 보시면, 사용자가 프롬프트를 던지면 keyword-detector가 받아서 상태를 기록하고, skill-injector가 그 상태를 보고 `<system-reminder>`를 Claude에게 주입한 다음, Claude가 도구를 쓰면 PostToolUse에서 verifier가 검증을 돌리는 흐름이에요. 28종 라이프사이클 이벤트가 있는데, OMC는 그중 핵심을 다 씁니다. 자주 보이는 신호는 `[MAGIC KEYWORD: autopilot]`, `The boulder never stops`, 그리고 메모리용 `<remember>` 태그(7일 TTL)예요. 비상시에는 `DISABLE_OMC=1`이나 `OMC_SKIP_HOOKS`로 끌 수 있다는 것도 기억해 두세요. 다음은 MCP와 상태, 팀 파이프라인을 한 슬라이드에 모았습니다.

## S8. OMC 깊이 보기 (4) MCP + .omc/ + Team

bridge 디렉터리는 esbuild로 번들된 .cjs 파일들이 모인 곳입니다. 메인 번들이 963KB인 `mcp-server.cjs`이고, 팀 인프라가 657KB인 `team-mcp.cjs`예요. 이 둘이 OMC의 두 다리입니다. 위쪽 트리 다이어그램에서 보시듯이 상태는 `.omc/state/sessions/{sessionId}/` 아래에 모드별로 JSON이 따로따로 떨어져 있어서, 한 머신에서 ralph랑 autopilot이랑 team을 동시에 돌려도 안 섞입니다. 아래 다이어그램이 Team 파이프라인인데요, plan, prd, exec, verify, fix 다섯 단계로 흐르고, fix는 bounded loop로 무한 반복을 막아 둡니다. 검증 통과하면 Done으로 빠져 나가고요. 이제 다른 결의 프레임워크인 Superpowers를 보겠습니다.

## S9. Superpowers — 가벼움이 곧 정체성

여기서 분위기를 한번 바꿔서, Superpowers를 보겠습니다. 같은 "큰 플러그인"인데 결이 완전히 다릅니다. 14개 스킬이 다인데요, `agents/`도, `commands/`도, MCP도, `src/`도, 빌드 파이프라인도 전부 없습니다. 훅도 SessionStart 딱 하나, 28종 중 1종만 씁니다. 행동의 무게는 전부 SKILL.md 본문이 짊어져요. 그런데도 Codex, Gemini, Cursor, OpenCode, Copilot CLI까지 매니페스트만 바꿔서 같은 마크다운을 재배포합니다. 이게 정말 강력한 부분이에요. 다음 슬라이드에서 OMC와 Superpowers를 한 표로 잘라 보겠습니다.

## S10. 비교 한 장 — 4 프레임워크

이 슬라이드 한 장에 차주 분석의 절반이 들어 있습니다. 위쪽 사분면 차트를 보시면 가로축은 가벼움-무거움, 세로축은 "오케스트레이션이 스킬 안인가 밖인가"예요. Superpowers는 좌측 하단(가벼움+스킬 안), OMC는 우측 상단(무거움+스킬 밖)에 가장 멀리 떨어져 있고, SuperClaude와 Official은 중간에 있습니다. 아래 표로 조립 단위, 상태, 멀티 에이전트 세 프레임을 잘라 봤어요. 가장 중요한 축은 위쪽 차트가 그대로 보여주는 "어디에 오케스트레이션이 사는가"입니다. 이 한 축이 무게랑 자유도랑 학습 비용을 거의 다 설명합니다. 이제 OMC와 Superpowers의 강점을 두 칼럼으로 비교해 봅시다.

## S11. 있는 것 vs 없는 것

그러면 OMC가 공짜로 주는 게 뭔지, Superpowers가 더 잘하는 게 뭔지 두 칼럼으로 나란히 비교해 봅시다. 왼쪽 OMC는 키워드 라우팅, 상태 보존, 검증 게이트, 팀 파이프라인 네 개를 줘요. 오른쪽 Superpowers는 멀티 하네스, 가벼움, 행동 평가 튜닝 세 개에서 더 강합니다. 한 줄로 정리하면 "OMC는 인프라, Superpowers는 콘텐츠"입니다. 우리가 어느 쪽으로 갈지는 환경에 따라 다른데, 일단 둘 다 깔고 키워드를 분리하는 것도 방법입니다. 이제 가장 실용적인 부분, 우리 손으로 작은 OMC를 따라 만드는 6단계로 넘어가겠습니다.

## S12. 나만의 OMC 만들기 — 6단계

여기서 가장 실용적인 부분으로 넘어가겠습니다. 위쪽 파이프라인 다이어그램을 보시면 Stage 1부터 6까지 화살표 하나로 이어져 있어요. 1단계는 hello 스킬 한 장, 2단계는 키워드 훅, 3단계는 첫 에이전트, 4단계는 자체 MCP, 5단계는 PRD 루프, 6단계는 미니 team입니다. 아래 표의 마지막 칼럼 "다음 트리거"가 핵심이에요. 욕구가 생겼을 때만 다음으로 갑니다. 처음부터 6단계 다 만들 필요 없어요. "키워드만 쳐도 떴으면"이라는 생각이 들면 그때 Stage 2로 가시면 됩니다. 다음은 이 6단계를 우리 일정에 어떻게 배치할지, 2주 학습 계획으로 가겠습니다.

## S13. 다음 2주 학습 계획

그래서 차주랑 차차주 계획을 이렇게 짰습니다. 이번 주는 휴일이라 1.5시간만 쓰고 큰 그림만 그리고요. 다음 주 9시간은 Superpowers, SuperClaude, OMC 코드를 직접 읽습니다. 특히 SuperClaude는 README 카운트만 인용했으니까 한 파일을 골라 100줄 요약하는 게 게이트예요. 차차주 10시간은 Stage 1, Stage 2까지 직접 만들어 보고요. 차차주 마지막 주말 게이트는 "키워드 트리거에 응답하는 hello 스킬"입니다. 다음은 우리가 이 길에서 만날 수 있는 흔한 실수 다섯 개를 미리 짚고 가겠습니다.

## S14. 흔한 실수 5개

여기서 잠깐, 미리 알면 시간 아끼는 실수 다섯 개만 짚고 갑니다. kebab-case 빠뜨리면 설치가 조용히 실패해요. namespacing 잊으면 표준 스킬이랑 충돌하고요. 디버깅 console.log가 stdout으로 새면 MCP stdio가 깨집니다. 도구 inputSchema 누락이랑 세션 ID 전역 캐시도 흔한 실수예요. 그리고 보너스로, description에 워크플로를 다 적어 버리면 모델이 본문을 안 읽습니다. 이건 Superpowers의 writing-skills SKILL이 행동 평가로 발견한 결론입니다. 자, 이제 발표 마무리로 우리 팀이 다음에 같이 해 볼 액션 세 가지를 제안드립니다.

## S15. 다음 액션

마지막으로 우리 팀이 다음에 시도해 볼 것 세 가지로 마무리하겠습니다. 첫째, 각자 1시간 들여서 Stage 1 hello 플러그인 한 명씩 만들어 봅시다. namespacing 감각이 생겨요. 둘째, OMC `scripts/keyword-detector.mjs`를 같이 읽는 시간을 잡으면 좋겠습니다. UserPromptSubmit 훅의 살아 있는 표본이에요. 셋째, 차차주 마지막 날 또는 그 다음 주 월요일에 Stage 2 데모로 자기 키워드 라우팅을 보여주는 자리 어떨까요. 레포는 vimkim/omc-code-analysis에 있고, REPORT.ko.md §6를 그대로 따라가시면 됩니다. 들어주셔서 감사합니다. 질문 받겠습니다.

## S16. 예상 질문 백업

**Q1. 우리 회사 환경에서 OMC 그대로 깔아도 되나요?**
네, npm 패키지로도 배포되니까 글로벌 설치 가능합니다. 다만 28종 hook 중 일부는 환경에 따라 침묵 실패할 수 있어서, 처음에는 `DISABLE_OMC=1`로 꺼 두고 들여다보다가 필요한 훅만 켜는 걸 권장드려요. 비상시에는 `OMC_SKIP_HOOKS=keyword-detector,skill-injector`처럼 콤마로 일부만 끌 수도 있습니다.

**Q2. Superpowers와 OMC 중 우리 팀에 뭐가 맞을까요?**
환경에 따라 답이 다릅니다. Codex나 Gemini 같은 다른 하네스도 같이 쓰신다면 Superpowers의 multi-harness가 강해요. 반대로 검증 게이트나 PRD 루프, 팀 파이프라인이 필요하면 OMC가 더 적합합니다. 그리고 둘은 배타적이지 않아서, 둘 다 깔고 키워드만 분리해도 됩니다. 어차피 SKILL.md는 호환 포맷이라 학습 자산이 안 사라집니다.

**Q3. SuperClaude는 왜 비교에만 짧게 들어갔나요?**
4.3.0 README의 "30 commands / 20 agents / 7 modes / 8 MCP" 카운트만 인용하고, 본격적인 코드 분석은 차주로 미뤘습니다. SuperClaude는 Python 인스톨러로 CLAUDE.md에 행동 주입을 깔고, 키워드로 페르소나가 자동 활성되는 결이라 OMC/Superpowers와 결이 다릅니다. 차주에 `src/`에서 한 페르소나 파일을 골라 100줄 요약하는 게 학습 게이트로 잡혀 있어요.
