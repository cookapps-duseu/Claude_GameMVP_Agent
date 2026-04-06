당신은 Game MVP Harness의 오케스트레이터입니다.
현재 진행 상황을 요약해서 출력하세요.

## 상태 파일 읽기

1. `artifacts/harness_state.md`를 읽으세요.
2. `artifacts/execution_plan.md`가 존재하면 읽어서 스프린트 완료 현황을 파악하세요.
3. `artifacts/token_log.md`가 존재하면 읽어서 토큰 집계를 계산하세요.

## 출력 형식

```
📊 Game MVP Harness 현재 상태
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
게임:         [output_folder 또는 "미정 (/game-init 실행 필요)"]
진행:         스프린트 [current_sprint] / [total_sprints]
재시도:       [retry_count]/3
기준 완화:    [relax_mode]
자동 진행:    [auto_proceed]
에스컬레이션: [escalation_count]회

마지막 커맨드: [last_command]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
스프린트 현황:
[execution_plan.md의 스텝 목록 — 완료: ✅, 미완료: [ ]]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💰 토큰 사용량 (token_log.md 기준, 서브에이전트 누적)
  Planner:              [합계]
  Generator (계약):     [합계]
  Generator (구현):     [합계]
  BrowserTester:        [합계]
  Evaluator:            [합계]
  ─────────────────────────────
  총합:                 [전체 합계] 토큰
  컨텍스트 대비:        [총합/200000*100]%  ← 호출당 200K 윈도우 기준
  (token_log.md가 없으면 이 섹션 생략)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
다음 커맨드: [next_command]
```

## 로그 파일 현황

`output/[output_folder]/log/` 폴더가 존재하면 어떤 단계 로그가 있는지 확인하고 출력:

```
📁 로그 파일 (output/[output_folder]/log/)
  init.md          ✅ / 없음
  sprint-1.md      ✅ / 없음   ← 있으면 재실행 시 재사용됨
  eval-1.md        ✅ / 없음
  sprint-2.md      ✅ / 없음
  ...
```

## harness_state.md가 없는 경우

```
❓ 아직 하네스가 초기화되지 않았습니다.
시작하려면: `/game-init`
```
