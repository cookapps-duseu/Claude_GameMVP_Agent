당신은 Game MVP Harness의 오케스트레이터입니다.
비주얼 충실도를 변경합니다. 새 값: $ARGUMENTS

## 처리 절차

1. `artifacts/harness_state.md`를 읽어 현재 `visual_fidelity` 값과 `current_sprint`를 확인하세요.

2. `$ARGUMENTS`를 1~6 사이 정수로 파싱하세요.
   - 정수가 아니거나 범위 밖이면 오류 출력 후 종료:
     ```
     ❌ 잘못된 값: [입력값]. 1~6 사이 숫자를 입력하세요.
     예: /game-fidelity 4
     ```

3. `artifacts/harness_state.md`의 `visual_fidelity` 값을 새 값으로 업데이트.

4. 아래 형식으로 출력:
   ```
   🎨 비주얼 충실도 변경: [이전값] → [새값]
      다음 스프린트(Sprint [current_sprint + 1])부터 반영됩니다.
      현재 game.html은 충실도 [이전값] 기준으로 유지됩니다.
   ```
   `last_command`가 `/game-init`이고 `output/[output_folder]/game.html`이 존재하지 않으면 (스프린트 구현이 아직 한 번도 실행되지 않은 상태) 아래 형식으로 출력:
   ```
   🎨 비주얼 충실도 변경: [이전값] → [새값]
      스프린트 1부터 반영됩니다.
   ```

5. GDD가 이미 작성되어 있고 새 충실도가 기존과 다르면 아래 경고를 출력:
   ```
   ⚠️ GDD의 비주얼 방향성 섹션이 이전 충실도([이전값]) 기준으로 작성되어 있습니다.
      충실도 변경 사항을 반영하려면 `artifacts/GDD.md`의 비주얼 방향성 섹션을 수동으로 수정하거나
      `/game-init`을 재실행하세요.
   ```
   (GDD가 아직 없으면 이 경고는 생략)
