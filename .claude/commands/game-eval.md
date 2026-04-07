당신은 Game MVP Harness의 오케스트레이터입니다.
검증할 스프린트 번호: $ARGUMENTS

## 시작 시 할 일 목록 생성

TodoWrite로 아래 태스크를 생성하세요:
1. 사전 확인 (harness_state, GDD, sprint_contract 읽기)
2. Eval 로그 파일 기록
3. Phase 1: Playwright 브라우저 테스트 직접 실행
4. Phase 2: Evaluator 실행
5. 결과 처리

각 태스크를 시작할 때 in_progress, 완료 시 completed로 표시하세요.

## 시간 추적 규칙

- **eval 시작 시각**을 기억해 두세요 (HH:MM:SS 형식).
- 각 Phase 시작 **직전**에도 시작 시각을 기록해 두세요.
- Phase 완료 후 경과 시간(분·초)을 계산하여 출력에 포함하세요.

## 토큰 로깅 및 출력 규칙

Evaluator 서브에이전트 완료 후:

1. `artifacts/token_log.md`에 행 추가:
```markdown
| [YYYY-MM-DD HH:MM] | Evaluator | 스프린트 $ARGUMENTS QA | [total_tokens 값] |
```

2. 아래 형식으로 대화창에 즉시 출력:
```
💰 토큰 현황 (Evaluator)
  이번 호출:    N,NNN 토큰
  스프린트 $ARGUMENTS 소계: token_log.md에서 이번 스프린트 행 합산
  전체 누적:    N,NNN 토큰  ([전체합/200000*100]% / 200K 기준)
  ⏱ 이번 단계: M분 SS초
```

서브에이전트 결과에서 `total_tokens: NNNNN` 값을 추출합니다. 없으면 "unknown"으로 기록하고 소계/누적에서 제외.

## 사전 확인

1. `artifacts/harness_state.md`를 읽어 `retry_count`, `pending_retry`, `output_folder`를 확인하세요.
2. `commands.md`를 읽어 현재 상황의 분기 규칙을 파악하세요.
3. `artifacts/sprint_contract.md`와 `artifacts/GDD.md`를 읽어 검증 기준을 파악하세요.

### pending_retry 상태 확인

`harness_state.md`의 `pending_retry: true`이면:
1. `artifacts/retry_feedback.md`를 읽어 누적 컨텍스트 파악
2. `harness_state.md` 업데이트: `retry_count: 0`, `pending_retry: false`
3. Generator 서브에이전트를 retry_feedback.md의 전체 컨텍스트와 함께 호출하여 수정
4. 수정 완료 후 Phase 1(브라우저 테스트)부터 재실행

## Eval 로그 파일 기록

`output/[output_folder]/log/eval-$ARGUMENTS.md`에 아래 내용을 저장하세요. (기존 파일이 있으면 덮어쓰기)

```markdown
# Eval $ARGUMENTS Plan Log
**생성일:** [YYYY-MM-DD HH:MM]
**game.html:** output/[output_folder]/game.html

## BrowserTester 시나리오
- game.html을 브라우저로 열어 GDD의 game_type 기반 조작 시뮬레이션 실행
- 콘솔 에러/경고 전체 캡처
- 단계별 스크린샷 저장: output/[output_folder]/screenshots/

## Evaluator 검증 기준 (sprint_contract.md 기반)
[artifacts/sprint_contract.md의 verifiable_criteria 항목 그대로 복사]

## retry_count
현재: [retry_count] / 6
```

## Phase 1: Playwright 브라우저 테스트 (직접 실행)

**서브에이전트 없이 오케스트레이터가 직접 Playwright MCP 도구를 사용합니다.**

### Step 1 — 게임 로드

`output/[output_folder]/` 폴더에 `game.html`이 있는지 확인하세요.
해당 폴더에서 로컬 HTTP 서버를 실행하거나, 아래 절대경로로 file:// URL을 구성하세요:

```
file:///e:/Github/Claude_GameMVP_Agent/output/[output_folder]/game.html
```

`browser_navigate` 도구로 위 URL을 여세요.
`browser_wait_for_timeout` 2000ms 대기.
`browser_screenshot` 으로 초기 화면 저장: `output/[output_folder]/screenshots/01-initial.png`

### Step 2 — GDD 기반 조작 시나리오

GDD의 `game_type`을 확인하여 아래 시나리오 중 해당하는 것을 실행하세요:

**tap/click 게임:**
1. 화면 중앙 `browser_click` (게임 시작)
2. `browser_screenshot` → `02-started.png`
3. 1초 간격으로 화면 중앙 10회 `browser_click`
4. `browser_screenshot` → `03-playing.png`
5. 게임오버 화면 감지 후 재시작 버튼 `browser_click` 시도
6. `browser_screenshot` → `04-restart.png`

**platformer/action 게임:**
1. `browser_press_key` Space 또는 Enter (게임 시작)
2. `browser_screenshot` → `02-started.png`
3. `browser_press_key` ArrowRight 3초 유지 → Space 3회 → ArrowLeft 1초
4. `browser_screenshot` → `03-playing.png`
5. 게임오버 후 재시작 키 입력
6. `browser_screenshot` → `04-restart.png`

**puzzle 게임:**
1. `browser_click` 화면 클릭 (시작)
2. `browser_screenshot` → `02-started.png`
3. 화면 내 요소를 좌→우→중앙 순으로 `browser_click` 3회 반복
4. `browser_screenshot` → `03-playing.png`
5. 레벨 클리어 또는 실패 후 다음 액션 클릭
6. `browser_screenshot` → `04-restart.png`

**기타/알 수 없음:**
- `browser_click` + `browser_press_key` 방향키 + Space 조합으로 진행

### Step 3 — 콘솔 로그 수집

`browser_console_messages` 도구로 콘솔 에러/경고 전체를 수집하세요.
error / warning / info 레벨로 구분하세요.

### Step 4 — browser_report.md 작성

`artifacts/browser_report.md`를 아래 형식으로 작성하세요:

```markdown
# Browser Test Report — Sprint $ARGUMENTS
**테스트일:** [YYYY-MM-DD]
**game.html 경로:** output/[output_folder]/game.html
**분석 방식:** Playwright MCP 실제 브라우저 실행

## 콘솔 에러/경고
| 레벨 | 메시지 | 소스/라인 |
|------|--------|-----------|
| error | [메시지] | [파일:라인] |

(없으면 "콘솔 에러 없음")

## 조작 로그
| 단계 | 액션 | 스크린샷 | 결과 |
|------|------|----------|------|
| 1 | 게임 로드 | 01-initial.png | 정상/비정상 |
| 2 | 게임 시작 | 02-started.png | 정상/비정상 |
| 3 | 플레이 | 03-playing.png | 정상/비정상 |
| 4 | 재시작 | 04-restart.png | 정상/비정상 |

## 크래시/멈춤
[없음 / 발생 시 단계와 증상 기술]

## 총평
[플레이 가능 여부, 주요 문제 1-3줄 요약]
```

browser_report.md 작성 완료 후 아래 형식으로 출력:
```
🌐 브라우저 테스트 완료 (스프린트 $ARGUMENTS)
콘솔 에러: N건
스크린샷: output/[folder]/screenshots/ (N장)
결과: 플레이 가능 / 크래시 / 에러 N건
⏱ 소요: M분 SS초
```

## Phase 2: Evaluator 서브에이전트 실행

`agents/evaluator.md`를 읽은 후 Evaluator 서브에이전트를 호출하세요.

Evaluator에게 전달할 컨텍스트:
- 스프린트 번호: $ARGUMENTS
- GDD.md 경로: artifacts/GDD.md
- sprint_contract.md 경로: artifacts/sprint_contract.md
- harness_state.md 경로: artifacts/harness_state.md
- browser_report.md 경로: artifacts/browser_report.md
- game.html 경로: output/[output_folder]/game.html

Evaluator가 `artifacts/qa_report.md`를 생성하면 즉시 토큰을 token_log.md에 기록하세요.

## 결과 분기

### PASS인 경우

1. `artifacts/harness_state.md` 업데이트:
   - `retry_count: 0`
   - `pending_retry: false`
   - `last_command: /game-eval $ARGUMENTS`
   - 마지막 스프린트 여부 확인 (`current_sprint == total_sprints`)

2. `artifacts/execution_plan.md`에서 QA 항목을 `[x]`로 갱신.

3. **마지막 스프린트가 아닌 경우:** 즉시 `/game-sprint [N+1]` 실행.

4. **마지막 스프린트인 경우:**

   `artifacts/token_log.md`를 읽어 전체 토큰 합산 후 아래 형식으로 출력:

   ```
   🎮 모든 스프린트 완료!
   최종 게임: output/[output_folder]/game.html
   데이터 파일: output/[output_folder]/*.csv

   💰 전체 토큰 사용량
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     Planner:         N,NNN 토큰
     Generator(계약): N,NNN 토큰
     Generator(구현): N,NNN 토큰
     Evaluator:       N,NNN 토큰
   ─────────────────────────────
     총합:            N,NNN 토큰
     200K 대비:       N.N%
     (200K × N.N회 분량)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ※ 브라우저 테스트는 오케스트레이터 직접 실행 (별도 토큰 없음)

   ⏱ 전체 소요 시간
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     시작: [첫 게임-init 또는 첫 스프린트 시작 시각 HH:MM]
     완료: [현재 시각 HH:MM]
     총합: H시간 M분
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   CSV 파일을 Excel에서 수정 후 브라우저를 새로고침하면 게임에 반영됩니다.
   ```

### FAIL — retry_count < 6인 경우

1. `artifacts/harness_state.md` 업데이트:
   - `retry_count += 1`
   - `next_command: /game-sprint $ARGUMENTS`

2. 실패 내용 출력:
   ```
   ❌ 스프린트 $ARGUMENTS FAIL (재시도 [retry_count]/6)
   실패 항목:
   - [criterion]: [description] — [location] ([severity])
   브라우저 에러: [콘솔 에러 요약]
   → Generator 수정 후 재검증 시작합니다.
   ```

3. Generator 서브에이전트 호출하여 수정:
   - `artifacts/browser_report.md`의 콘솔 에러 전문 전달
   - `artifacts/qa_report.md`의 실패 기준 + 재현 방법 전달
   - 크래시 직전 스크린샷 경로 전달

4. Generator 수정 완료 후 Phase 1(브라우저 테스트)부터 즉시 재실행.

### FAIL — retry_count >= 6인 경우

1. `artifacts/retry_feedback.md` 생성/갱신:

```
# Retry Feedback — Sprint $ARGUMENTS
**상태:** pending_retry
**마지막 시도:** [YYYY-MM-DD]
**누적 시도 횟수:** [retry_count]

## 미해결 문제 목록
[qa_report.md의 failures 항목별로]
- [criterion]: [증상] — [game.html 위치] ([severity])
  - 시도한 수정 이력: [각 retry에서 Generator가 시도한 접근법]
  - 실패 원인 추정: [왜 수정이 안 됐는지]

## 브라우저 에러 로그 (최신)
[browser_report.md의 콘솔 에러 전문 복사]

## 스크린샷 경로
[browser_report.md의 스크린샷 목록 복사]

## 다음 재시도 시 Generator에게 전달할 컨텍스트
[qa_report.md의 failures 전문 복사]

## harness 복원 정보
sprint: $ARGUMENTS
retry_count: 0
pending_retry: false
```

2. `artifacts/harness_state.md` 업데이트:
   - `pending_retry: true`
   - `next_command: /game-sprint $ARGUMENTS`

3. 사용자에게 안내 후 대기 (사용자 판단 필요):
   ```
   ⚠️ 6회 재시도 후에도 미해결 문제가 남았습니다.

   미해결 항목:
   - [실패 기준]: [증상] ([심각도])

   누적 피드백이 artifacts/retry_feedback.md에 저장되었습니다.
   나중에 "재시도해줘"라고 하면 이어서 진행합니다.
   현재 game.html은 output/[output_folder]/에 저장되어 있습니다.
   ```
