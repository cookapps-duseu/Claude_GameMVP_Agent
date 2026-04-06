# Game MVP Agent

게임 컨셉 문서 3종을 입력하면 플레이 가능한 게임 HTML을 자율 생성하는 Claude Code 기반 에이전트 하네스.

---

## 어떻게 동작하나요?

3개의 에이전트가 순서대로 실행됩니다.

```
Planner → Generator → BrowserTester → Evaluator
           ↑________________↓ (FAIL 시 최대 6회 반복)
```

| 에이전트 | 역할 |
|----------|------|
| **Planner** | 컨셉 3종 분석 → GDD + 스프린트 실행 플랜 생성 |
| **Generator** | 스프린트 단위로 game.html + CSV 데이터 파일 구현 |
| **BrowserTester** | Playwright로 실제 브라우저에서 게임 실행 및 조작 테스트 |
| **Evaluator** | 브라우저 테스트 결과 + 코드 분석으로 QA 채점 |

---

## 시작 전 준비

### 1. 의존성 설치

```bash
npm install -g @playwright/mcp
```

### 2. Claude Code 설정

`.claude/settings.json`에 Playwright MCP가 이미 등록되어 있습니다.
Claude Code를 재시작하면 자동으로 연결됩니다.

### 3. 게임 컨셉 파일 3종 준비

| 파일 | 내용 |
|------|------|
| `concept.html` | 게임 전체 컨셉 + 이미지/분위기 |
| `core_fun.md` | 무엇이 플레이어를 즐겁게 하는가 |
| `core_loop.md` | 입력→행동→결과→피드백 사이클 |

---

## 사용 방법

### 게임 만들기

Claude Code 대화창에 입력:

```
[폴더 경로] 기반으로 게임 만들어줘
```

예시:
```
C:/Users/me/my-game-concept 기반으로 게임 만들어줘
```

**Edit Automatically** 모드를 켜두면 완성까지 자동으로 실행됩니다.

---

### 진행 중 사용할 수 있는 명령

| 입력 | 동작 |
|------|------|
| `이어서 해줘` | 멈춘 곳부터 재개 |
| `상태 알려줘` | 현재 진행 상황 + 다음 단계 확인 |
| `재시도해줘` | 6회 실패 후 누적 피드백으로 재시도 |

---

## 결과물

게임 완성 시 `output/YYYY-MM-DD-게임이름/` 폴더에 생성됩니다.

```
output/2024-01-01-MyGame/
├── game.html          ← 브라우저에서 바로 실행
├── levels.csv         ← 레벨 데이터
├── enemies.csv        ← 적 데이터
├── items.csv          ← 아이템 데이터
└── log/               ← 단계별 실행 로그 (재시작용)
    ├── init.md
    ├── sprint-1.md
    ├── eval-1.md
    └── ...
```

`game.html`을 브라우저로 열면 바로 플레이할 수 있습니다.
CSV 파일을 Excel에서 수정한 뒤 브라우저를 새로고침하면 게임 수치가 바뀝니다.

---

## 중단 후 재시작

컴퓨터를 껐다 켜거나 대화창을 닫아도 이어서 할 수 있습니다.

```
이어서 해줘
```

특정 스프린트부터 다시 시작하고 싶으면:

```
/game-sprint 2
```

`log/sprint-2.md` 파일이 있으면 기존 계약을 재사용합니다.
새로 계약을 작성하려면 해당 파일을 삭제 후 재실행하세요.

---

## 슬래시 커맨드 (직접 실행)

| 커맨드 | 설명 |
|--------|------|
| `/game-init [경로]` | 최초 실행 — 컨셉 파일 수집 → Planner 실행 |
| `/game-sprint N` | 스프린트 N 구현 |
| `/game-eval N` | 스프린트 N QA 검증 (브라우저 테스트 포함) |
| `/game-status` | 현재 진행 상태 확인 |

---

## 프로젝트 구조

```
├── CLAUDE.md                # 에이전트 동작 규칙
├── commands.md              # 커맨드 레지스트리
├── agents/
│   ├── planner.md           # Planner 시스템 프롬프트
│   ├── generator.md         # Generator 시스템 프롬프트
│   ├── evaluator.md         # Evaluator 시스템 프롬프트
│   └── browser_tester.md   # BrowserTester 시스템 프롬프트
├── artifacts/               # 에이전트 간 통신 파일
│   ├── harness_state.md     # 현재 진행 상태
│   ├── GDD.md               # 게임 디자인 문서 (Planner 생성)
│   ├── execution_plan.md    # 전체 실행 플랜
│   ├── sprint_contract.md   # 현재 스프린트 계약
│   ├── browser_report.md    # 브라우저 테스트 결과
│   └── qa_report.md         # QA 결과
└── output/
    └── YYYY-MM-DD-게임이름/
        ├── game.html
        ├── *.csv
        └── log/
```
