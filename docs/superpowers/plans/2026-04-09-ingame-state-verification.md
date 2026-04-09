# 인게임 상태 전환 버그 방지 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generator가 상태 전환 패턴을 올바르게 구현하도록 강제하고, game-eval이 인게임 진입 후 입력 불능 버그를 자동 감지하도록 한다.

**Architecture:** agents/generator.md에 상태 전환 코드 패턴을 "반드시 준수" 규칙으로 추가하고, .claude/commands/game-eval.md의 Step 3 시작 부분과 각 game_type 플레이 액션 직후에 `window.__debug.stats.stateChanges` / `eventLog` 기반 검증 블록을 삽입한다.

**Tech Stack:** Markdown 편집만 사용. 테스트는 실제 game-eval 실행으로 확인.

---

## 파일 변경 목록

| 파일 | 변경 유형 | 내용 |
|------|---------|------|
| `agents/generator.md` | 수정 | "반드시 준수" 섹션에 상태 전환 패턴 추가 + 자기 체크리스트 2항목 추가 |
| `.claude/commands/game-eval.md` | 수정 | Step 3 시작부에 인게임 전환 확인 블록 추가 + 각 game_type 블록 플레이 후에 입력 반응 확인 블록 추가 |

---

### Task 1: generator.md — 상태 전환 패턴 강제 규칙 추가

**Files:**
- Modify: `agents/generator.md`

- [ ] **Step 1: "반드시 준수" 섹션에 상태 전환 패턴 블록 삽입**

`agents/generator.md`에서 아래 텍스트를 찾는다:

```
**CSV 로드 패턴 (반드시 사용):**
```

그 바로 앞에 아래 블록을 삽입한다:

```markdown
- **상태 전환 패턴 (반드시 준수):** 게임은 outgame / ingame / gameover 3개 상태를 명확히 분리해야 한다. 각 상태 전환 시 반드시 아래 순서를 따른다:
  1. 이전 상태의 이벤트 리스너 제거 (`removeEventListener`)
  2. `cancelAnimationFrame`으로 게임 루프 중단
  3. 게임 변수 초기화
  4. 새 상태의 이벤트 리스너 등록
  5. `requestAnimationFrame` 루프 재시작

  **필수 패턴:**
  ```javascript
  let currentState = 'outgame';
  let animFrameId = null;

  function cleanup() {
    if (animFrameId) { cancelAnimationFrame(animFrameId); animFrameId = null; }
    // 이전 상태 이벤트 리스너 제거 (addEventListener로 등록한 것을 removeEventListener로 제거)
  }

  function enterState(newState) {
    var prev = currentState;
    cleanup();
    currentState = newState;
    window.__debug.stats.stateChanges.push({ from: prev, to: newState, timestamp: Date.now() });
    if (newState === 'ingame') initGame();
    else if (newState === 'gameover') showGameOver();
    else if (newState === 'outgame') showTitle();
  }
  ```

```

- [ ] **Step 2: 자기 체크리스트에 2항목 추가**

`agents/generator.md`에서 아래 줄을 찾는다:

```
- [ ] 핵심 로직(공격/데미지/상태 전이)에 __debug 로깅 호출이 삽입되어 있는가
```

그 바로 뒤에 아래 2줄을 추가한다:

```markdown
- [ ] 상태 전환 시 이전 이벤트 리스너를 제거하는가 (`removeEventListener` 사용)
- [ ] 인게임 진입 시 게임 루프가 새로 시작되는가 (`requestAnimationFrame` 재호출)
```

- [ ] **Step 3: 변경 결과 확인**

`agents/generator.md`를 읽어 아래를 확인:
- "상태 전환 패턴 (반드시 준수)" 섹션이 "CSV 로드 패턴" 앞에 존재하는가
- `enterState` 함수 패턴이 포함되어 있는가
- 체크리스트에 "removeEventListener", "requestAnimationFrame 재호출" 두 항목이 있는가

- [ ] **Step 4: 커밋**

```bash
git add agents/generator.md
git commit -m "feat: generator에 상태 전환 패턴 강제 규칙 추가"
```

---

### Task 2: game-eval.md — Step 3 인게임 활성화 검증 추가

**Files:**
- Modify: `.claude/commands/game-eval.md`

- [ ] **Step 1: Step 3 시작부에 인게임 전환 확인 블록 추가**

`.claude/commands/game-eval.md`에서 아래 텍스트를 찾는다:

```
GDD의 `game_type`에 따라 인간형 시나리오 실행:

**tap/click 게임:**
```

그 사이 (`인간형 시나리오 실행:` 다음 줄, `**tap/click 게임:**` 앞)에 아래 블록을 삽입한다:

```markdown
**[인게임 전환 확인] 플레이 액션 전 — 모든 game_type 공통:**

`browser_evaluate`로 아래를 실행한다:

```javascript
(function() {
  if (typeof window.__debug === 'undefined') return { skip: true };
  var changes = window.__debug.stats.stateChanges;
  var inIngame = changes.some(function(s) { return s.to === 'ingame'; });
  return { inIngame: inIngame, stateChanges: changes };
})()
```

- `skip: true` → window.__debug 없음, 이 검증 skip
- `inIngame === false` → **[인게임 전환 실패]** 즉시 수정 루프 트리거
  - 에러 메시지: `"[인게임 전환 실패] stateChanges에 to:'ingame' 없음 — enterState('ingame') 호출 누락 또는 상태 명칭 불일치"`

---

```

- [ ] **Step 2: tap/click 게임 블록 — 플레이 후 입력 반응 확인 추가**

`.claude/commands/game-eval.md`에서 아래 텍스트를 찾는다 (tap/click 블록 내부):

```
2. 7~12회 랜덤 횟수로 요소 ref 기반 클릭 (1~2초 랜덤 간격)
3. **실시간 디버그 감지:**
```

그 사이에 아래 블록을 삽입한다:

```markdown
3. **[인게임 입력 반응 확인] 플레이 액션 후:**
   `browser_evaluate`로 아래를 실행한다:
   ```javascript
   (function() {
     if (typeof window.__debug === 'undefined') return { skip: true };
     var inIngame = window.__debug.stats.stateChanges.some(function(s) { return s.to === 'ingame'; });
     var eventCount = window.__debug.stats.eventLog.length;
     return { inIngame: inIngame, eventCount: eventCount };
   })()
   ```
   - `inIngame === true && eventCount === 0` → **[인게임 입력 불능]** 즉시 수정 루프 트리거
     - 에러 메시지: `"[인게임 입력 불능] ingame 전환 후 eventLog 비어있음 — 이벤트 리스너 미등록 또는 게임 루프 미시작"`
   - `inIngame === false` → 이미 Step 1에서 처리됨, skip
```

기존 번호는 한 칸씩 뒤로 밀린다 (기존 3→4, 4→5, 5→6):

```markdown
4. **실시간 디버그 감지:**
   ...
5. `browser_console_messages(level: "warning")` 수집
6. `browser_take_screenshot` → `03-playing.png`
```

- [ ] **Step 3: platformer/action 게임 블록 — 플레이 후 입력 반응 확인 추가**

`.claude/commands/game-eval.md`에서 아래 텍스트를 찾는다 (platformer/action 블록 내부):

```
5. `browser_press_key` ArrowLeft 3~8회 반복 (각 입력 사이 100~300ms 딜레이)
6. **실시간 디버그 감지:**
```

그 사이에 아래 블록을 삽입한다:

```markdown
6. **[인게임 입력 반응 확인] 플레이 액션 후:**
   `browser_evaluate`로 아래를 실행한다:
   ```javascript
   (function() {
     if (typeof window.__debug === 'undefined') return { skip: true };
     var inIngame = window.__debug.stats.stateChanges.some(function(s) { return s.to === 'ingame'; });
     var eventCount = window.__debug.stats.eventLog.length;
     return { inIngame: inIngame, eventCount: eventCount };
   })()
   ```
   - `inIngame === true && eventCount === 0` → **[인게임 입력 불능]** 즉시 수정 루프 트리거
     - 에러 메시지: `"[인게임 입력 불능] ingame 전환 후 eventLog 비어있음 — 이벤트 리스너 미등록 또는 게임 루프 미시작"`
```

기존 번호는 한 칸씩 뒤로 밀린다 (기존 6→7, 7→8, 8→9):

```markdown
7. **실시간 디버그 감지:**
   ...
8. `browser_console_messages(level: "warning")` 수집
9. `browser_take_screenshot` → `03-playing.png`
```

- [ ] **Step 4: puzzle 게임 블록 — 플레이 후 입력 반응 확인 추가**

`.claude/commands/game-eval.md`에서 아래 텍스트를 찾는다 (puzzle 블록 내부):

```
2. 3~5회 임의 순서로 ref 기반 클릭 (딜레이 포함)
3. **실시간 디버그 감지:**
```

그 사이에 아래 블록을 삽입한다:

```markdown
3. **[인게임 입력 반응 확인] 플레이 액션 후:**
   `browser_evaluate`로 아래를 실행한다:
   ```javascript
   (function() {
     if (typeof window.__debug === 'undefined') return { skip: true };
     var inIngame = window.__debug.stats.stateChanges.some(function(s) { return s.to === 'ingame'; });
     var eventCount = window.__debug.stats.eventLog.length;
     return { inIngame: inIngame, eventCount: eventCount };
   })()
   ```
   - `inIngame === true && eventCount === 0` → **[인게임 입력 불능]** 즉시 수정 루프 트리거
     - 에러 메시지: `"[인게임 입력 불능] ingame 전환 후 eventLog 비어있음 — 이벤트 리스너 미등록 또는 게임 루프 미시작"`
```

기존 번호는 한 칸씩 뒤로 밀린다 (기존 3→4, 4→5, 5→6):

```markdown
4. **실시간 디버그 감지:**
   ...
5. `browser_console_messages(level: "warning")` 수집
6. `browser_take_screenshot` → `03-playing.png`
```

- [ ] **Step 5: 변경 결과 확인**

`.claude/commands/game-eval.md`를 읽어 아래를 확인:
- Step 3 시작부(game_type 분기 전)에 "인게임 전환 확인" 블록이 있는가
- tap/click, platformer/action, puzzle 블록 각각에 "인게임 입력 반응 확인" 블록이 있는가
- 에러 메시지 2종(`[인게임 전환 실패]`, `[인게임 입력 불능]`)이 명시되어 있는가
- 기존 실시간 디버그 감지 블록이 그대로 유지되어 있는가

- [ ] **Step 6: 커밋**

```bash
git add .claude/commands/game-eval.md
git commit -m "feat: game-eval Step 3에 인게임 활성화 검증 추가"
```
