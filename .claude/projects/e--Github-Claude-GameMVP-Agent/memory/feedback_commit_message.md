---
name: 커밋 메시지 언어 규칙
description: git commit 메시지 작성 시 한국어 사용 규칙
type: feedback
---

커밋 타입 접두사(feat:, fix:, refactor:, docs:, chore: 등)는 영어 유지, 그 뒤 설명은 한국어로 작성.

예시:
- `feat: 비주얼 충실도 시스템 추가`
- `fix: game-fidelity 스프린트 시작 조건 개선`
- `refactor: 충실도 질문을 game-sprint에서 game-init으로 이동`

**Why:** 사용자 선호. 타입 접두사는 컨벤션상 영어 유지.
**How to apply:** 모든 git commit -m 작성 시 설명 부분을 한국어로.
