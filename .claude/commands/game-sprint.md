당신은 Game MVP Harness의 오케스트레이터입니다.
실행할 스프린트 번호: $ARGUMENTS

## 시작 시 할 일 목록 생성

TodoWrite로 아래 태스크를 생성하세요:
1. 사전 확인 (harness_state, GDD, commands 읽기)
2. 계약 작성 또는 로그 파일 로드
3. 계약 자체 검토 (verifiability, scope, hardcode)
4. 구현 (Generator IMPLEMENT)
5. 완료 처리 및 game-eval 실행

각 태스크를 시작할 때 in_progress, 완료 시 completed로 표시하세요.

## 사전 확인

1. `artifacts/harness_state.md`를 읽어 `output_folder`, `retry_count` 등 현재 상태를 확인하세요.
2. `artifacts/GDD.md`를 읽어 스프린트 $ARGUMENTS의 목표와 verifiable_criteria를 확인하세요.
3. `commands.md`를 읽어 현재 상황에 맞는 다음 커맨드를 파악하세요.

## 시간 추적 규칙

- **스프린트 시작 시각**을 기억해 두세요 (HH:MM:SS 형식).
- 각 서브에이전트 호출 **직전**에도 시작 시각을 기록해 두세요.
- 호출 완료 후 경과 시간(분·초)을 계산하여 출력에 포함하세요.

## 토큰 로깅 및 출력 규칙

모든 서브에이전트 호출이 완료될 때마다:

1. `artifacts/token_log.md`에 행 추가:
```markdown
| [YYYY-MM-DD HH:MM] | Generator | 스프린트 $ARGUMENTS [CONTRACT/IMPLEMENT] | [total_tokens 값] |
```

2. 아래 형식으로 대화창에 즉시 출력:
```
💰 토큰 현황 (Generator [CONTRACT/IMPLEMENT])
  이번 호출:       N,NNN 토큰  (이 호출 하나의 컨텍스트 N.N% 사용)
  스프린트 $ARGUMENTS 소계: N,NNN 토큰
  전체 누적:       N,NNN 토큰  (호출당 200K 윈도우 기준 누적 N.N%)
  ⏱ 이번 단계:    M분 SS초
```
※ 컨텍스트 대비 %는 호출 1회 200K 기준이며 월 사용량과 무관합니다.

서브에이전트 결과에서 `total_tokens: NNNNN` 값을 추출합니다. 없으면 "unknown"으로 기록하고 소계/누적에서 제외.

## Phase A: 계약 작성

### 로그 파일 확인

`output/[output_folder]/log/sprint-$ARGUMENTS.md`가 존재하는지 확인하세요.

**파일이 존재하는 경우 (재시작/재사용):**
- 해당 파일을 읽어 계약 내용으로 사용
- `artifacts/sprint_contract.md`를 해당 내용으로 덮어쓰기
- Generator CONTRACT 단계를 건너뜀
- 출력:
  ```
  📂 기존 계약 로그 발견: output/[output_folder]/log/sprint-$ARGUMENTS.md
  해당 파일 내용으로 진행합니다. 새로 작성하려면 파일을 삭제 후 재실행하세요.
  ```

**파일이 존재하지 않는 경우 (최초 실행):**

`agents/generator.md`를 읽은 후 Generator 서브에이전트를 **CONTRACT 모드**로 호출하세요.

Generator에게 전달할 컨텍스트:
- 모드: "CONTRACT"
- 스프린트 번호: $ARGUMENTS
- GDD.md 경로: artifacts/GDD.md
- harness_state.md 경로: artifacts/harness_state.md

Generator(CONTRACT) 완료 후 즉시 토큰을 token_log.md에 기록하세요.

`artifacts/sprint_contract.md` 작성이 완료되면, 동일한 내용을 `output/[output_folder]/log/sprint-$ARGUMENTS.md`에도 저장하세요. (폴더가 없으면 생성)

계약 내용을 아래 형식으로 출력하세요:
```
📋 스프린트 $ARGUMENTS 계약
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
목표: [스프린트 목표]
구현 항목:
- [항목 1]
검증 기준:
- [기준 1]
CSV 파일: [파일명 목록]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Phase A-1: 계약 자체 검토

아래 3가지 기준으로 `artifacts/sprint_contract.md`를 검토하세요.

**기준 1 — verifiable_criteria 모호성**
- FAIL: "타격감이 좋아야 한다" / PASS: "적 클릭 시 화면이 0.1초 흔들린다"
- 모호한 기준 1개 이상 → 재작성 요청

**기준 2 — GDD 정합성**
- GDD 스프린트 $ARGUMENTS 목표와 계약 구현 항목 대조
- 빠진 기능 또는 추가된 기능(스코프 크리프) → 재작성 요청

**기준 3 — 하드코딩 위험**
- 수치/파라미터가 포함된 기능이 있으면 해당 값이 CSV 컬럼으로 정의되어 있는지 확인
- CSV 없이 HTML 하드코딩 가능성 → 재작성 요청

문제 있으면 Generator에게 수정 지시 후 재검토 (최대 2회). 문제 없으면 즉시 Phase B 진행.

## Phase B: 구현

Generator 서브에이전트를 **IMPLEMENT 모드**로 호출하세요.

Generator에게 전달할 컨텍스트:
- 모드: "IMPLEMENT"
- 스프린트 번호: $ARGUMENTS
- GDD.md 경로: artifacts/GDD.md
- sprint_contract.md 경로: artifacts/sprint_contract.md
- harness_state.md 경로: artifacts/harness_state.md
- qa_report.md 경로: artifacts/qa_report.md (이전 FAIL 내용이 있으면)

Generator(IMPLEMENT) 완료 후 즉시 토큰을 token_log.md에 기록하세요.

## 완료 처리

1. `artifacts/harness_state.md` 업데이트:
   - `last_command: /game-sprint $ARGUMENTS`
   - `next_command: /game-eval $ARGUMENTS`

2. `artifacts/execution_plan.md`에서 해당 스프린트 항목을 `[x]`로 갱신.

3. 스프린트 전체 소요 시간을 출력:
   ```
   ⏱ 스프린트 $ARGUMENTS 총 소요: M분 SS초  (HH:MM 시작 → HH:MM 완료)
   ```

4. 즉시 `/game-eval $ARGUMENTS` 실행.
