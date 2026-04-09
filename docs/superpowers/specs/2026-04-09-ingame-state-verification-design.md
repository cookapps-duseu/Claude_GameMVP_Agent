# 인게임 상태 전환 버그 방지 설계

## 목표

Generator가 생성한 game.html에서 아웃게임→인게임 전환 후 플레이가 작동하지 않는 버그를 예방하고 자동 감지한다.

---

## 변경 범위

| 파일 | 변경 유형 | 내용 |
|------|---------|------|
| `agents/generator.md` | 수정 | 상태 전환 패턴 강제 규칙 + 자기 체크리스트 항목 추가 |
| `.claude/commands/game-eval.md` | 수정 | Phase 1 Step 3에 인게임 활성화 검증 추가 |

---

## 섹션 1: Generator 상태 전환 패턴 강화

### 배경

Generator가 아웃게임→인게임 전환 시 이벤트 리스너 관리, 게임 루프 제어, 상태 초기화를 제대로 처리하지 않아 인게임에서 입력이 동작하지 않는 버그가 발생한다.

### 규칙 추가 위치

`agents/generator.md`의 "반드시 준수" 섹션에 추가.

### 추가 내용

**상태 전환 패턴 (반드시 준수):**

게임은 아래 3개 상태를 명확히 분리해야 한다:
- `outgame`: 타이틀/메뉴 화면
- `ingame`: 실제 플레이 화면
- `gameover`: 게임오버/결과 화면

각 상태 전환 시 반드시 아래 순서를 따른다:
1. 이전 상태의 이벤트 리스너 제거 (`removeEventListener`)
2. `cancelAnimationFrame`으로 게임 루프 중단
3. 게임 변수 초기화
4. 새 상태의 이벤트 리스너 등록
5. `requestAnimationFrame` 루프 재시작 (또는 이벤트 기반 렌더링 시작)

**상태 전환 함수 필수 패턴:**

```javascript
let currentState = 'outgame';
let animFrameId = null;

function cleanup() {
  if (animFrameId) cancelAnimationFrame(animFrameId);
  // 이전 상태 이벤트 리스너 제거
}

function enterState(newState) {
  const prev = currentState;
  cleanup();
  currentState = newState;
  window.__debug.stats.stateChanges.push({ from: prev, to: newState, timestamp: Date.now() });
  if (newState === 'ingame') initGame();
  else if (newState === 'gameover') showGameOver();
  else if (newState === 'outgame') showTitle();
}
```

### 자기 체크리스트 추가 항목

- [ ] 상태 전환 시 이전 이벤트 리스너를 제거하는가 (`removeEventListener` 사용)
- [ ] 인게임 진입 시 게임 루프가 새로 시작되는가 (`requestAnimationFrame` 재호출)

---

## 섹션 2: game-eval 인게임 활성화 검증

### 배경

game-eval Phase 1이 시각적 이상(흰 화면, UI 없음)만 감지하고, 인게임 화면은 보이지만 입력이 실제로 동작하지 않는 케이스를 잡지 못한다.

### 검증 위치

Phase 1 Step 3 (플레이) 내부:
- 플레이 액션 **전**: 인게임 전환 여부 확인
- 플레이 액션 **후**: 입력 반응 여부 확인

### 검증 로직

**검증 1 — 인게임 전환 확인 (플레이 액션 전):**

```javascript
(function() {
  var changes = window.__debug.stats.stateChanges;
  var inIngame = changes.some(function(s) { return s.to === 'ingame'; });
  return { inIngame: inIngame, stateChanges: changes };
})()
```

- `inIngame === false` → **"인게임 전환 실패"** → 즉시 수정 루프 트리거

**검증 2 — 인게임 입력 반응 확인 (플레이 액션 후):**

```javascript
(function() {
  var eventCount = window.__debug.stats.eventLog.length;
  var inIngame = window.__debug.stats.stateChanges.some(function(s) { return s.to === 'ingame'; });
  return { inIngame: inIngame, eventCount: eventCount };
})()
```

- `inIngame === true && eventCount === 0` → **"인게임 진입됐으나 입력 불능"** → 즉시 수정 루프 트리거

### 에러 메시지 형식

즉시 수정 루프 진입 시 아래 메시지를 에러로 기록:

- 케이스 1: `[인게임 전환 실패] stateChanges에 to:'ingame' 없음 — enterState('ingame') 호출 누락 또는 상태 명칭 불일치`
- 케이스 2: `[인게임 입력 불능] ingame 전환 후 eventLog 비어있음 — 이벤트 리스너 미등록 또는 게임 루프 미시작`

---

## 안전장치

| 상황 | 처리 |
|------|------|
| `window.__debug` 없음 | 검증 skip, 기존 시각적 감지만 사용 |
| game_type이 이벤트를 발생시키지 않는 구조 | eventLog 검증은 skip, stateChanges 검증만 수행 |
| 10회 수정 후에도 미해결 | browser_report에 기록 후 Phase 2(Evaluator)로 진행 |
