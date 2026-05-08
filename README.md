# omc-code-analysis

큰 Claude Code 플러그인이 어떻게 만들어지는가에 대한 코드 분석 노트입니다.
oh-my-claudecode (OMC) 내부 구조, Superpowers / SuperClaude / Anthropic 공식 플러그인 시스템을 비교한 자료를 모아 두었습니다.

교육 목적으로 정리되었습니다 — 저자도 학습자입니다.

## 파일 구성

```
omc-code-analysis/
├── README.md                                  # 이 문서
├── REPORT.ko.md                               # 메인 보고서 (한국어, 정제본)
└── research/
    ├── plugin-frameworks-internal-omc.md      # OMC 내부 분석 자료 (raw)
    ├── plugin-frameworks-external.md          # Superpowers / SuperClaude / 공식 분석 자료 (raw)
    └── grill-and-revise-log.json              # 반복 리뷰 로그 (3 라운드, REVISE→REVISE→APPROVED)
```

- **`REPORT.ko.md`** — 한국어 정제본. 다음 9개 섹션으로 구성됩니다.
  1. Claude Code 플러그인의 해부도
  2. oh-my-claudecode 깊이 읽기 (패키징, Skills, Agents, Hooks, MCP, src/, .omc/, 빌드, 행동 패턴)
  3. Superpowers 깊이 읽기
  4. OMC vs Superpowers vs SuperClaude vs 공식 비교 (4개 표)
  5. 있는 것과 없는 것의 차이
  6. 나만의 OMC 만들기 — 6단계 학습 로드맵
  7. 다음 2주 학습 계획
  8. 부록 (인용 출처)

- **`research/plugin-frameworks-internal-omc.md`** — `oh-my-claudecode` 리포지토리를 직접 읽고 정리한 raw 자료. 파일 경로와 줄 번호 인용 위주.

- **`research/plugin-frameworks-external.md`** — `obra/superpowers`, `SuperClaude_Framework`, Anthropic 공식 플러그인 문서를 외부에서 조사한 raw 자료. URL 인용 위주.

- **`research/grill-and-revise-log.json`** — 보고서를 작성한 라이터 / 비평가 두 에이전트의 반복 검토 로그. 1라운드(20개 이슈 → REVISE), 2라운드(5개 이슈 → REVISE), 3라운드(0개 이슈 → APPROVED).

## 작성 방식

이 보고서는 [grill-and-revise](https://github.com/Yeachan-Heo/oh-my-claudecode) 패턴으로 작성되었습니다 — Claude Code 안에서 라이터(executor) 에이전트와 비평가(critic) 에이전트를 분리해, 비평가가 사실 / 깊이 / 구조 / 톤 / 길이 / 인용 6개 카테고리로 검수하고, 비평가가 명시적으로 `VERDICT: APPROVED`를 출력할 때까지 라이터가 다시 쓰는 루프입니다.

총 3 라운드, 라운드 캡 4. 라운드별 이슈 수와 카테고리는 `research/grill-and-revise-log.json`에 남아 있습니다.

## 다루지 않는 것

- 본 보고서는 OMC를 **읽고** 무엇을 배웠는지를 정리한 학습 노트입니다. OMC 자체의 사용법 매뉴얼이 아닙니다 — 사용법은 https://github.com/Yeachan-Heo/oh-my-claudecode 본 리포지토리를 보세요.
- Anthropic 공식 플러그인 스펙의 100% 망라가 아닙니다. 출처가 두 raw 자료에 있는 사실로만 한정했습니다.

## 라이선스

학습 노트이며, 인용된 모든 코드/문서의 원저작권은 각 프로젝트에 귀속됩니다.
- oh-my-claudecode: MIT (Yeachan Heo)
- Superpowers: MIT (Jesse Vincent)
- SuperClaude_Framework: 해당 리포 라이선스
- Anthropic 공식 문서: https://code.claude.com 약관

본 분석 노트 자체는 MIT.
