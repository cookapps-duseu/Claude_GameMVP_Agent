# 커맨드 레지스트리

모든 에이전트는 작업 시작 전 이 파일을 읽고,
완료 시 아래 표를 기준으로 다음 커맨드를 안내해야 합니다.

## 상황별 커맨드

| 상황 | 커맨드 | 설명 |
|------|--------|------|
| 최초 시작 | `/game-init` | 입력 파일 수집 → Planner → 실행 플랜 생성 |
| 스프린트 구현 시작 | `/game-sprint N` | Generator 실행 (계약 작성 → 구현) |
| 구현 완료 후 | `/game-eval N` | Evaluator QA 실행 |
| QA FAIL → 수정 재시도 | `/game-sprint N` | retry_count 증가, FAIL 항목 수정 |
| retry_count >= 6 | (대기) | retry_feedback.md 저장, 사용자 재시도 대기 |
| 사용자 "재시도해줘" | `/game-sprint N` | retry_feedback.md 읽어 retry_count 0으로 리셋 |
| QA PASS | `/game-sprint N+1` | 다음 스프린트 진행 |
| 마지막 스프린트 PASS | (종료) | output/YYYY-MM-DD-GameTitle/ 폴더 확인 |
| 현재 상태 모를 때 | `/game-status` | 상태 요약 + 다음 커맨드 안내 |

## 에이전트별 다음 커맨드 규칙

### Planner 완료 후
→ `/game-sprint 1`

### Generator (계약 모드) 완료 후
→ 사용자 계약 승인 대기 (auto_proceed=true이면 즉시 구현 모드 진행)

### Generator (구현 모드) 완료 후
→ `/game-eval N`

### Evaluator — PASS
→ 마지막 스프린트: 완료 선언
→ 그 외: `/game-sprint N+1`

### Evaluator — FAIL (retry_count < 6)
→ harness_state의 retry_count += 1 후 `/game-sprint N`

### Evaluator — FAIL (retry_count >= 6)
→ `artifacts/retry_feedback.md` 생성/갱신 (누적 피드백 저장)
→ harness_state의 next_command: /game-sprint N, pending_retry: true 기록
→ 사용자에게 안내 후 대기:
   "⚠️ 6회 재시도 후에도 미해결 문제가 남았습니다.
    피드백이 artifacts/retry_feedback.md에 저장되었습니다.
    나중에 '재시도해줘'라고 하면 이어서 진행합니다."

### pending_retry 상태에서 재시도
→ retry_feedback.md 읽어 누적 컨텍스트 확인
→ harness_state의 retry_count: 0, pending_retry: false 리셋
→ `/game-sprint N` 실행 (누적 피드백을 Generator에 전달)

## harness_state.md 필드 정의

```yaml
current_sprint: 1          # 현재 스프린트 번호
total_sprints: 4           # 전체 스프린트 수 (GDD에서)
retry_count: 0             # 현재 스프린트 FAIL 재시도 횟수 (PASS 시 0으로 리셋)
relax_mode: false          # true이면 Evaluator 기준 완화 적용
auto_proceed: true         # true이면 승인 프롬프트 생략
escalation_count: 0        # 자동 에스컬레이션 횟수 (2 이상이면 강제 중단)
pending_retry: false       # true이면 retry_feedback.md 대기 상태
last_command: /game-init
next_command: /game-sprint 1
output_folder: ""          # Planner가 채움 (예: 2026-04-03-DragonSlayer)
```
