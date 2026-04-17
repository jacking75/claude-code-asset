# 프로젝트: Next.js + Three.js 브라우저 게임

## 기술 스택
- **프레임워크**: Next.js 14 (App Router)
- **3D 렌더링**: Three.js r160+
- **물리 엔진**: Rapier.js (WASM 기반)
- **언어**: TypeScript (strict mode)
- **상태 관리**: Zustand

## 프로젝트 구조
- `src/app/` - Next.js 페이지 및 라우팅
- `src/game/` - 게임 핵심 로직 (Three.js 씬, 엔티티 등)
- `src/game/entities/` - 플레이어, 적, NPC 클래스
- `src/game/physics/` - Rapier 물리 세계 관리
- `src/game/assets/` - GLTF 모델, 텍스처 로더
- `src/hooks/` - React 커스텀 훅 (useGameLoop, useThreeScene 등)
- `public/models/` - GLTF/GLB 3D 에셋

## 코딩 규칙
- Three.js 씬은 반드시 `useEffect` cleanup에서 `dispose()` 호출
- `requestAnimationFrame` 루프는 컴포넌트 언마운트 시 취소
- GLTF 모델은 `GLTFLoader` + `DRACOLoader`로 압축 로드
- 모바일 성능을 위해 `InstancedMesh` 우선 사용
- 물리 바디 생성 시 반드시 cleanup에서 `world.removeRigidBody()` 호출

## Three.js 작업 시 참고
- 최신 Three.js API 확인: https://threejs.org/docs/
- 공식 예제: https://threejs.org/examples/

## 커뮤니티 제작 Three.js 게임 개발 스킬 설치
```
npx skillfish add natea/fitfinder threejs-game
```  