---
name: threejs-scene
description: >
  Three.js 씬(Scene) 구성 및 WebGL 렌더링 설정. 새 씬 생성, 카메라 설정,
  조명 추가, 렌더러 초기화, 애니메이션 루프 구성 시 자동 적용.
  Next.js와 통합할 때 SSR 이슈 방지 패턴 포함.
allowed-tools: Read Grep Glob
---

# Three.js Scene 설정 플레이북

## Next.js에서 Three.js 초기화 필수 패턴

Three.js는 브라우저 전용 API(WebGL, window)를 사용하므로 반드시 클라이언트 컴포넌트로 선언해야 합니다.

```typescript
'use client'; // ← 반드시 필요!

import { useEffect, useRef } from 'react';
import * as THREE from 'three';

export default function GameCanvas() {
  const mountRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!mountRef.current) return;

    // 1. 씬 생성
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x87ceeb);

    // 2. 카메라 (게임용 원근 카메라)
    const camera = new THREE.PerspectiveCamera(
      75,
      mountRef.current.clientWidth / mountRef.current.clientHeight,
      0.1,
      1000
    );
    camera.position.set(0, 5, 10);

    // 3. 렌더러 설정 (성능 최적화 포함)
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // 모바일 최적화
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    mountRef.current.appendChild(renderer.domElement);

    // 4. 애니메이션 루프
    let animId: number;
    const animate = () => {
      animId = requestAnimationFrame(animate);
      renderer.render(scene, camera);
    };
    animate();

    // 5. 리사이즈 핸들러
    const handleResize = () => {
      if (!mountRef.current) return;
      camera.aspect = mountRef.current.clientWidth / mountRef.current.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(mountRef.current.clientWidth, mountRef.current.clientHeight);
    };
    window.addEventListener('resize', handleResize);

    // 6. 반드시 cleanup!
    return () => {
      cancelAnimationFrame(animId);
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
      mountRef.current?.removeChild(renderer.domElement);
    };
  }, []);

  return <div ref={mountRef} style={{ width: '100%', height: '100vh' }} />;
}
```

## 조명 설정 (게임 표준)
- `AmbientLight`: 전체 환경광 (강도 0.3~0.5)
- `DirectionalLight`: 태양광, 그림자 맵 활성화
- `HemisphereLight`: 하늘/땅 색상 혼합, 실외 씬에 효과적

## 성능 최적화 규칙
- `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` 항상 적용
- 동일 지오메트리 반복 사용 시 `InstancedMesh` 필수
- 씬에 추가된 모든 geometry, material은 `dispose()` 호출