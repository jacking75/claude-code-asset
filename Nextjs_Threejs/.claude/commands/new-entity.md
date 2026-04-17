```markdown
---
description: 새 게임 엔티티 클래스 생성 (플레이어/적/NPC)
allowed-tools: Read, Write, Bash
---

`$ARGUMENTS` 이름의 새 게임 엔티티를 생성합니다.

1. `src/game/entities/` 디렉토리 구조 확인
2. 기존 엔티티 패턴 참고 (`src/game/entities/` 내 파일 검토)
3. `$ARGUMENTS.ts` 파일 생성 (TypeScript, GameEntity 상속)
4. 해당 엔티티를 `src/game/entities/index.ts`에 export 추가
5. 기본 테스트 파일 `src/game/entities/__tests__/$ARGUMENTS.test.ts` 생성