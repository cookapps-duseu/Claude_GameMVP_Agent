# Game MVP Harness

게임 컨셉 문서 3종을 입력받아 플레이 가능한 게임 MVP HTML을 자율 생성하는 3-에이전트 하네스.

## 자동 트리거 규칙

**사용자가 아래와 같은 말을 하면 즉시 해당 커맨드를 실행하세요. 별도 확인 없이 바로 실행합니다.**

| 사용자 발화 패턴 | 자동 실행 커맨드 |
|-----------------|----------------|
| "MVP 만들어줘", "게임 만들어줘", "게임 개발해줘", "만들어줘" + 폴더/파일 언급 | `/game-init [언급된 경로]` |
| "이 폴더로 만들어줘", "여기 있는 파일로 만들어줘", "[경로] 기반으로 만들어줘" | `/game-init [해당 경로]` |
| "지금 어디야", "어디까지 했어", "상태 알려줘", "진행 상황" | `/game-status` |
| "계속해줘", "다음 진행해줘", "이어서 해줘" | `harness_state.md`의 `next_command` 실행 |
| "재시도해줘", "다시 시도해줘" + `pending_retry: true` 상태 | `harness_state.md` 확인 후 `/game-eval N` 실행 (retry_feedback.md 컨텍스트 포함) |

경로가 메시지에 포함된 경우 그 경로를 인자로 사용합니다.
경로가 없으면 `/game-init`을 인자 없이 실행합니다.

## 시작하기

처음 실행하는 경우:
```
/game-init [컨셉 파일이 있는 폴더 경로]
```

현재 진행 상황을 모르는 경우:
```
/game-status
```

## 에이전트 구조

| 에이전트 | 역할 | 프롬프트 파일 |
|----------|------|---------------|
| Planner | 입력 3종 분석 → GDD + 실행 플랜 생성 | `agents/planner.md` |
| Generator | GDD 기반 스프린트 단위 game.html 구현 | `agents/generator.md` |
| Evaluator | 스프린트 산출물 QA 검증 + 피드백 | `agents/evaluator.md` |

## 슬래시 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/game-init` | 최초 실행: 입력 파일 수집 → Planner → 실행 플랜 생성 |
| `/game-sprint N` | 스프린트 N 구현 (Generator 실행) |
| `/game-eval N` | 스프린트 N QA 검증 (Evaluator 실행) |
| `/game-status` | 현재 진행 상태 + 다음 커맨드 안내 |

## 파일 구조

```
├── commands.md              # 커맨드 레지스트리 (언제 어떤 커맨드를 쓰는지)
├── agents/                  # 에이전트 시스템 프롬프트
├── artifacts/               # 에이전트 간 통신 파일
│   ├── harness_state.md     # 진행 상태 추적
│   ├── GDD.md               # Planner 산출물
│   ├── execution_plan.md    # 전체 실행 플랜
│   ├── sprint_contract.md   # 스프린트 계약
│   ├── qa_request.md        # QA 요청
│   └── qa_report.md         # QA 결과
└── output/
    └── YYYY-MM-DD-GameTitle/
        ├── game.html
        ├── levels.csv
        ├── enemies.csv
        ├── items.csv
        └── config.csv
```

## auto_proceed 모드

`artifacts/harness_state.md`의 `auto_proceed: true`로 설정하면
모든 "진행할까요?" 승인 프롬프트를 생략하고 자동으로 다음 단계를 실행합니다.
Claude Code의 "Edit Automatically" 설정과 함께 사용하면 완전 자율 실행됩니다.
