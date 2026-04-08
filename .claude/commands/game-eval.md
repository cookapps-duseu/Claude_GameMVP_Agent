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

> **사전 준비:** `artifacts/harness_state.md`에서 읽은 `output_folder` 값을 이후 모든 경로의 `[output_folder]` 플레이스홀더에 치환하여 사용합니다.

**서브에이전트 없이 오케스트레이터가 직접 Playwright MCP 도구를 사용합니다.**

### 인간형 조작 원칙

모든 클릭은 아래 방식으로 수행합니다. 하드코딩된 좌표(예: 항상 400,300) 사용을 금지합니다.

**버튼 클릭 — DOM 스냅샷 우선:**
1. `browser_snapshot`으로 클릭 가능한 요소(button, a, [role=button], canvas 등) 목록 확인
2. 클릭 대상 요소의 `ref` 값과 `element` 설명을 추출
3. `browser_click(ref=..., element=...)`으로 클릭

**스크린샷 fallback (DOM으로 못 찾을 경우):**
- 스크린샷을 시각적으로 판단하여 버튼처럼 보이는 영역 특정
- `browser_snapshot`에서 가장 가까운 ref를 찾아 클릭

**키 입력:**
- 연속 키 입력 사이 50~200ms 랜덤 딜레이
- 방향키 유지 시간: 1.5~3.5초 랜덤

---

### 에러 감지 기준

각 step 완료 후 아래 세 조건 중 하나라도 해당하면 즉시 수정 트리거:

**A. 콘솔 에러/warning:** `browser_console_messages`에서 level이 `error` 또는 `warning`인 항목 1건 이상

**B. 스크린샷 이상:**
- 화면이 흰색/단색으로만 채워져 있음
- 게임 UI 요소가 보이지 않음
- 화면이 이전 step과 동일 (진행 없음)

**C. Step 결과 비정상:**
- 게임로드 후 시작 버튼/화면이 없음
- 시작 액션 후 게임 화면으로 전환 안 됨
- 플레이 중 캐릭터/요소가 화면에 없음
- 재시작 후 여전히 게임오버 화면

---

### 즉시 수정 루프

에러 감지 시 아래 루프를 실행합니다 (step당 최대 10회):

```
step_retry_count = 0

loop:
  step_retry_count += 1
  
  if step_retry_count > 10:
    step_error_log에 기록 → 다음 step으로 강제 진행
    break
  
  1. game.html 읽기 (Read 도구)
  2. 에러 메시지 + 스크린샷에서 원인 파악
  3. game.html 해당 부분 직접 수정 (Edit 도구) — 매 시도마다 다른 접근법
  4. browser_navigate로 새로고침
  5. 해당 step 재실행
  6. 에러 감지 재확인 → 에러 없으면 break
```

**수정 원칙:**
- 에러 메시지가 있으면 그 에러를 직접 수정
- 스크린샷 이상이면 렌더링/초기화 코드 확인
- 같은 수정을 반복하지 않음 — 매 시도마다 다른 접근법

---

### Step 1 — 게임 로드

`output/[output_folder]/` 폴더에 `game.html`이 있는지 확인합니다.
아래 file:// URL을 구성합니다:

```
file:///e:/Github/Claude_GameMVP_Agent/output/[output_folder]/game.html
```

1. `browser_navigate`로 위 URL을 엽니다.
2. 2000ms 대기.
3. `browser_console_messages(level: "warning")` 수집.
4. `browser_take_screenshot` → `output/[output_folder]/screenshots/01-initial.png`

**에러 감지 후 수정 루프 실행.**

step 1이 10회 초과 실패 시: 나머지 step 전부 skip, browser_report에 "로드 실패" 기록 후 Phase 2로 진행.

---

### Step 2 — 게임 시작

GDD의 `game_type`을 확인하여 시작 방식 결정:

**tap/click / puzzle 게임:**
1. `browser_snapshot`으로 시작 버튼 탐색
2. 클릭 대상 요소의 `ref` 값 추출 후 `browser_click(ref=..., element=...)`으로 클릭
3. `browser_console_messages(level: "warning")` 수집
4. `browser_take_screenshot` → `02-started.png`

**platformer/action 게임:**
1. `browser_press_key` Space 또는 Enter
2. `browser_console_messages(level: "warning")` 수집
3. `browser_take_screenshot` → `02-started.png`

**에러 감지 후 수정 루프 실행.**

---

### Step 3 — 플레이

GDD의 `game_type`에 따라 인간형 시나리오 실행:

**tap/click 게임:**
1. `browser_snapshot`으로 클릭 가능 요소 탐색
2. 7~12회 랜덤 횟수로 요소 ref 기반 클릭 (1~2초 랜덤 간격)
3. `browser_console_messages(level: "warning")` 수집
4. `browser_take_screenshot` → `03-playing.png`

**platformer/action 게임:**
1. `browser_press_key` ArrowRight 5~12회 반복 (각 입력 사이 100~300ms 딜레이)
2. 딜레이 50~200ms
3. `browser_press_key` Space 2~4회 (각 입력 사이 딜레이 포함)
4. 딜레이 50~200ms
5. `browser_press_key` ArrowLeft 3~8회 반복 (각 입력 사이 100~300ms 딜레이)
6. `browser_console_messages(level: "warning")` 수집
7. `browser_take_screenshot` → `03-playing.png`

**puzzle 게임:**
1. `browser_snapshot`으로 클릭 가능 요소 탐색
2. 3~5회 임의 순서로 ref 기반 클릭 (딜레이 포함)
3. `browser_console_messages(level: "warning")` 수집
4. `browser_take_screenshot` → `03-playing.png`

**기타/알 수 없음:**
- `browser_snapshot` 탐색 후 임의 클릭 + 방향키 + Space 조합

**에러 감지 후 수정 루프 실행.**

---

### Step 4 — 재시작

Step 3 이후 스크린샷/snapshot에서 게임오버·재시작 버튼 요소가 감지되지 않으면 Step 4를 skip합니다.

게임오버 화면 감지 후 재시작 액션:

1. `browser_snapshot`으로 재시작 버튼/요소 탐색
2. 대상 요소의 `ref` 값 추출 후 `browser_click(ref=..., element=...)`으로 클릭 (없으면 Space/Enter/R 키 시도)
3. `browser_console_messages(level: "warning")` 수집
4. `browser_take_screenshot` → `04-restart.png`

**에러 감지 후 수정 루프 실행.**

---

### Step 5 — browser_report.md 작성

모든 step 완료 후 `artifacts/browser_report.md`를 작성합니다:

```markdown
# Browser Test Report — Sprint [N]
**테스트일:** [YYYY-MM-DD]
**game.html 경로:** output/[output_folder]/game.html
**분석 방식:** Playwright MCP 실제 브라우저 실행

## 콘솔 에러/경고
| 레벨 | 메시지 | 소스/라인 |
|------|--------|-----------|
| error | [메시지] | [파일:라인] |

(없으면 "콘솔 에러 없음")

## 조작 로그
| Step | 액션 | 스크린샷 | 결과 | 수정 횟수 |
|------|------|----------|------|---------|
| 1. 게임로드 | browser_navigate | 01-initial.png | 정상/비정상 | N회 |
| 2. 게임시작 | [액션] | 02-started.png | 정상/비정상 | N회 |
| 3. 플레이 | [시나리오] | 03-playing.png | 정상/비정상 | N회 |
| 4. 재시작 | [액션] | 04-restart.png | 정상/비정상 | N회 |

## 수정 이력 (Phase 1 즉시 수정)

| Step | 시도 횟수 | 에러 내용 | 수정 내용 | 결과 |
|------|---------|---------|---------|------|
| [Step명] | [N] | [에러 요약] | [수정 내용] | 해결/미해결 |

(수정 없이 통과한 step은 생략)

## 크래시/멈춤
[없음 / 발생 시 step과 증상 기술]

## 총평
[플레이 가능 여부, 주요 문제 1-3줄 요약]
```

browser_report.md 작성 완료 후 아래 형식으로 출력:
```
🌐 브라우저 테스트 완료 (스프린트 [N])
콘솔 에러: N건
스크린샷: output/[folder]/screenshots/ (N장)
즉시 수정: N회 (해결 M건 / 미해결 K건)
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

#### 자가 수복 트리거 확인

FAIL 처리 전, 아래 두 조건 중 하나를 확인하세요:

**조건 A — 반복 FAIL 감지:**
`artifacts/repair_history.md`가 존재하는 경우, 이번 스프린트에서 동일한 FAIL 항목이 2회 이상 반복되었는지 확인합니다.
(qa_report.md의 실패 criterion과 repair_history.md의 이전 기록 비교)

**조건 B — 사용자 명시 요청:**
이 eval이 "룰 검토해줘", "에이전트 고쳐줘", "qa 룰이 이상해" 발화로 트리거된 경우.

**조건 충족 시 → 자가 수복 실행 (아래 "자가 수복 루틴" 참고)**
**조건 미충족 시 → 기존 Generator 수정 루틴 실행:**

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

---

### 자가 수복 루틴

자가 수복 트리거 조건이 충족된 경우 아래 순서로 실행합니다.

#### Phase R0 — 시작 알림

```
🛠 자가 수복 시작 — Sprint $ARGUMENTS
트리거: [반복 FAIL 감지 / 사용자 요청]
수복 에이전트를 실행합니다...
```

#### Phase R1 — SelfRepairDiagnoser 실행

`agents/self_repair_diagnoser.md`를 읽은 후 SelfRepairDiagnoser 서브에이전트를 호출합니다.

전달 컨텍스트:
- 스프린트 번호: $ARGUMENTS
- `artifacts/browser_report.md`
- `artifacts/qa_report.md`
- `artifacts/repair_history.md` (없으면 없다고 알림)
- `artifacts/GDD.md`
- `artifacts/sprint_contract.md`
- `agents/evaluator.md`
- `.claude/commands/game-eval.md`

SelfRepairDiagnoser가 `artifacts/repair_diagnosis.md`를 생성하면 진행합니다.

원인 유형이 **GAME_BUG**이면:
- 자가 수복 없이 기존 Generator 수정 루틴으로 즉시 fallback
- (위 "FAIL — retry_count < 6인 경우"의 기존 루틴 실행)

#### Phase R2 — SelfRepairFixer 실행

원인 유형이 GAME_BUG가 아닌 경우:
`agents/self_repair_fixer.md`를 읽은 후 SelfRepairFixer 서브에이전트를 호출합니다.

전달 컨텍스트:
- `artifacts/repair_diagnosis.md`
- 수정 대상 파일 경로

SelfRepairFixer 완료 후:
- `artifacts/harness_state.md`의 `self_repair_count` += 1

#### Phase R3 — 재검증 (Phase 1부터 재실행)

수복 후 Phase 1(Playwright 브라우저 테스트)부터 즉시 재실행합니다.

**결과 비교 (두 단계):**

① **Phase 1 정상화 확인**: 새 browser_report.md에 크래시/도구 오류가 있는가?
② **FAIL 항목 수 비교**: 수복 전 qa_report.md의 failures 수 vs 수복 후 failures 수

**악화 판정 조건** (둘 중 하나):
- Phase 1 도구 오류가 새로 발생
- FAIL 항목 수가 수복 전보다 증가

**악화 시 — 백업 복원:**
```
⚠️ 수복 결과 악화 감지 — 백업 복원 중
[수정 파일명]: artifacts/repair_backup/[파일명].bak 복원
→ GAME_BUG 루틴으로 fallback합니다.
```
`artifacts/repair_backup/[파일명].bak`에서 원본 파일 복원 후 기존 Generator 수정 루틴 실행.
`artifacts/repair_history.md`의 마지막 행 결과 컬럼을 "실패(복원)"으로 갱신.

**개선 시 — 수복 성공:**
```
✅ 자가 수복 성공 — Sprint $ARGUMENTS
수정 파일: [파일명]
개선: [수복 전 N개] → [수복 후 M개] FAIL 항목
→ 파이프라인 계속 진행합니다.
```
`artifacts/repair_history.md`의 마지막 행 결과 컬럼을 "성공"으로 갱신.

#### Phase R4 — 영구화 질문 (완료 시점)

아래 두 시점 중 하나에서, `artifacts/repair_history.md`에 이번 세션 수복 이력이 있는 경우:
- 모든 스프린트 완료 시
- 단독 `/game-eval` 완료 시 (중간 재시작)

아래 메시지를 출력하고 사용자 응답을 기다립니다:

```
🔧 이번 세션에서 수복된 룰이 있습니다:
- [파일명]: [수정 내용 요약] ([일시])

영구 반영할까요? (git commit)
Y → 수복된 파일을 커밋하고 artifacts/repair_backup/ 삭제
N → 백업 복원 (원본 룰 유지, game.html은 현재 상태 유지)
```

**Y인 경우:**
1. 수복된 파일 git add + git commit (메시지: "fix: self-repair — [수정 내용 요약]")
2. `artifacts/repair_backup/` 폴더 삭제

**N인 경우:**
1. `artifacts/repair_backup/` 의 모든 .bak 파일로 원본 복원
2. 복원 완료 알림:
   ```
   ↩️ 원본 룰 복원 완료
   이번 게임의 수복은 임시 적용이었습니다.
   game.html은 현재 상태로 유지됩니다.
   ```

---

### 중간 재시작 감지

`/game-eval $ARGUMENTS`가 단독으로 실행된 경우 (전체 파이프라인이 아닌 사용자 직접 실행):

eval 완료 후, `harness_state.md`의 `auto_proceed: false`이면:

```
✅ 스프린트 $ARGUMENTS eval 완료.

남은 단계가 있습니다:
[execution_plan.md에서 미완료 스프린트 목록]

전체를 이어서 진행할까요?
Y (또는 "계속해줘") → 다음 스프린트부터 파이프라인 끝까지 자동 진행
N → 현재 단계만 실행하고 대기
```

`auto_proceed: true`이면 이 질문 없이 즉시 다음 스프린트 진행.

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
