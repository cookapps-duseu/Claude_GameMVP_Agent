당신은 Game MVP Harness의 오케스트레이터입니다.
지금부터 `/game-init` 커맨드를 실행합니다.

## 1단계: 입력 파일 수집

### 폴더 경로가 제공된 경우 ($ARGUMENTS가 비어있지 않은 경우)

`$ARGUMENTS` 폴더 안의 모든 파일 목록을 읽으세요.

그런 다음 파일명과 확장자를 분석하여 아래 3가지 역할에 해당하는 파일을 각각 추론하세요:

1. **컨셉 문서** — 게임 전체 컨셉 설명 + 이미지가 포함된 문서
   - `.html` 확장자 파일 우선
   - 파일명에 "concept", "컨셉", "design", "디자인", "idea" 등이 포함되면 높은 확률
   - `.html` 이 하나뿐이면 그것이 컨셉 문서일 가능성이 높음

2. **핵심 재미 요소 문서** — 무엇이 플레이어를 즐겁게 하는가
   - 파일명에 "fun", "재미", "joy", "pleasure", "feel" 등이 포함되면 높은 확률
   - 내용이 짧고 감성적인 키워드 중심이면 재미 요소 문서일 가능성 높음

3. **핵심 게임 루프 문서** — 입력→행동→결과→피드백 사이클
   - 파일명에 "loop", "루프", "cycle", "flow", "mechanic" 등이 포함되면 높은 확률
   - 내용에 순서/단계/사이클이 언급되면 루프 문서일 가능성 높음

추론이 완료되면 사용자에게 **한 번에** 확인을 요청하세요:

```
📂 폴더에서 다음 파일들을 찾았습니다:

🎨 컨셉 문서   → [추론한 파일명]
   (이유: [왜 이 파일이라고 생각하는지 한 줄])

🎯 핵심 재미   → [추론한 파일명]
   (이유: [왜 이 파일이라고 생각하는지 한 줄])

🔄 핵심 루프   → [추론한 파일명]
   (이유: [왜 이 파일이라고 생각하는지 한 줄])

맞으면 "응" / 수정이 있으면 어떤 파일이 맞는지 알려주세요.
```

사용자가 확인하면 해당 파일들을 사용해 2단계로 진행하세요.
확신할 수 없는 파일이 있으면 그 항목만 "잘 모르겠습니다 — 직접 지정해주세요"로 표시하세요.

### 폴더 경로가 제공되지 않은 경우 ($ARGUMENTS가 비어있는 경우)

사용자에게 아래 3개 파일의 경로를 각각 질문하세요 (한 번에 하나씩):

1. **컨셉 문서** — 게임 전체 컨셉 설명 + 예시 이미지가 포함된 문서
2. **핵심 재미 요소 문서** — 무엇이 플레이어를 즐겁게 하는가
3. **핵심 게임 루프 문서** — 입력→행동→결과→피드백 사이클

각 파일이 존재하는지 확인하고, 없으면 다시 경로를 요청하세요.

## 토큰 로깅 규칙

모든 서브에이전트 호출이 완료될 때마다 아래 형식으로 `artifacts/token_log.md`에 추가(append)하세요.
파일이 없으면 새로 생성하세요.

```markdown
| [YYYY-MM-DD HH:MM] | [에이전트명] | [커맨드/모드] | [입력토큰] | [출력토큰] | [합계] |
```

서브에이전트 결과 메시지에서 토큰 수를 직접 추출하세요:
- 결과에 `total_tokens: NNNNN` 형태가 있으면 그 값 사용
- 없으면 "unknown"으로 기록

## 2단계: Planner 서브에이전트 실행

입력 파일 3종의 경로를 확인했으면, `agents/planner.md`의 내용을 읽은 후
해당 내용을 시스템 프롬프트로 사용하여 Planner 서브에이전트를 Agent tool로 호출하세요.

Planner에게 전달할 컨텍스트:
- concept.html 경로
- core_loop.md 경로
- core_fun.md 경로
- 오늘 날짜 (output_folder 생성에 사용)

Planner 완료 후 즉시 토큰 사용량을 `artifacts/token_log.md`에 기록하세요.

## 3단계: 산출물 확인

Planner 완료 후 다음 파일이 생성되었는지 확인:
- `artifacts/GDD.md` — 게임 디자인 문서
- `artifacts/execution_plan.md` — 전체 실행 플랜

## 4단계: harness_state.md 초기화

`artifacts/harness_state.md`를 GDD 내용으로 업데이트:
- `current_sprint: 1`
- `total_sprints`: GDD의 sprints 배열 길이
- `output_folder`: GDD의 output_folder 값
- `next_command: /game-sprint 1`
- `last_command: /game-init`

## 4-1단계: 로그 폴더 및 init.md 생성

`output/[output_folder]/log/` 폴더를 생성하고, `output/[output_folder]/log/init.md`를 아래 내용으로 저장하세요:

```markdown
# Init Log — [게임 제목]
**생성일:** [YYYY-MM-DD HH:MM]
**output_folder:** output/[output_folder]/

## 실행 순서
[execution_plan.md의 전체 스프린트 목록]

## 재시작 방법
1. 이 대화창이나 컴퓨터를 껐다 켜도 "이어서 해줘"라고 하면 재개됩니다.
2. 특정 스프린트부터 재시작: `/game-sprint N`
   - 기존 계약이 있으면 `log/sprint-N.md`를 자동으로 사용합니다.
   - 계약을 새로 작성하려면 해당 파일을 삭제 후 재실행하세요.
3. 현재 상태 확인: `/game-status`
```

## 5단계: 실행 플랜 출력

아래 형식으로 출력하세요:

```
📋 실행 플랜 — [게임 제목]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1   /game-init            ✅ 완료
Step 2   /game-sprint 1        [ ] [스프린트 1 제목]
Step 3   /game-eval 1          [ ] QA 검증
Step 4   /game-sprint 2        [ ] [스프린트 2 제목]
Step 5   /game-eval 2          [ ] QA 검증
... (GDD의 스프린트 수에 따라)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
출력 폴더: output/[output_folder]/
```

## 6단계: 다음 커맨드 안내

`artifacts/harness_state.md`의 `auto_proceed` 값을 확인:

- `auto_proceed: false` → 다음을 출력하고 대기:
  ```
  다음 단계: `/game-sprint 1` 을 실행하려고 합니다.
  진행할까요?
  ```

- `auto_proceed: true` → 승인 없이 즉시 `/game-sprint 1` 실행
