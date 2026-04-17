---
name: game-entity
description: >
  게임 엔티티(플레이어, 적, NPC, 아이템) 클래스 생성. Three.js Object3D 기반
  엔티티 시스템, GLTF 모델 로드, 애니메이션 믹서 설정 시 사용.
allowed-tools: Read Grep Glob
---

# 게임 엔티티 시스템

## 기본 엔티티 클래스 패턴

```typescript
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

export abstract class GameEntity {
  mesh: THREE.Group | null = null;
  mixer: THREE.AnimationMixer | null = null;
  actions: Map<string, THREE.AnimationAction> = new Map();

  abstract update(delta: number): void;

  async loadModel(path: string, scene: THREE.Scene): Promise<void> {
    const loader = new GLTFLoader();
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('/draco/'); // public/draco/ 필요
    loader.setDRACOLoader(dracoLoader);

    const gltf = await loader.loadAsync(path);
    this.mesh = gltf.scene;
    this.mixer = new THREE.AnimationMixer(this.mesh);

    gltf.animations.forEach((clip) => {
      this.actions.set(clip.name, this.mixer!.clipAction(clip));
    });

    scene.add(this.mesh);
  }

  playAnimation(name: string, fadeTime = 0.3): void {
    const action = this.actions.get(name);
    if (!action) return;
    this.actions.forEach((a) => a.fadeOut(fadeTime));
    action.reset().fadeIn(fadeTime).play();
  }

  dispose(scene: THREE.Scene): void {
    if (this.mesh) {
      scene.remove(this.mesh);
      this.mesh.traverse((obj) => {
        if (obj instanceof THREE.Mesh) {
          obj.geometry.dispose();
          if (Array.isArray(obj.material)) {
            obj.material.forEach((m) => m.dispose());
          } else {
            obj.material.dispose();
          }
        }
      });
    }
  }
}
```