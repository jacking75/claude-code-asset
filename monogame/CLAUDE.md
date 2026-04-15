# MonoGame 프로젝트 전역 지침

## 프로젝트 개요
이 프로젝트는 MonoGame 프레임워크(C#/.NET)를 사용하는 게임 개발 프로젝트입니다.

## 기술 스택
- **Framework**: MonoGame 3.8.4+ (Microsoft.Xna.Framework 기반)
- **Language**: C# (.NET 9 이상)
- **IDE**: Visual Studio Code / Rider / Visual Studio
- **Content Pipeline**: MGCB (MonoGame Content Builder)

## 핵심 아키텍처 원칙
1. MonoGame의 Game 루프(Initialize → LoadContent → Update → Draw)를 엄수한다.
2. GameComponent / DrawableGameComponent 패턴으로 기능을 분리한다.
3. ContentManager를 통해 에셋을 로드하고, Dispose 패턴을 준수한다.
4. SpriteBatch.Begin/End 쌍을 항상 맞춘다.
5. GameTime을 활용해 프레임 독립적 업데이트를 구현한다.

## 코딩 컨벤션
- 클래스명: PascalCase (예: PlayerSprite, GameStateManager)
- 필드명: _camelCase (예: _spriteBatch, _graphicsDevice)
- 상수명: UPPER_SNAKE_CASE (예: SCREEN_WIDTH, MAX_ENEMIES)
- 모든 IDisposable 구현체는 using 블록 또는 명시적 Dispose 호출

## 주요 네임스페이스
- `Microsoft.Xna.Framework` — 핵심 클래스 (Game, GameTime, Color, Vector2 등)
- `Microsoft.Xna.Framework.Graphics` — 렌더링 (SpriteBatch, Texture2D, Effect 등)
- `Microsoft.Xna.Framework.Input` — 입력 (Keyboard, Mouse, GamePad)
- `Microsoft.Xna.Framework.Audio` — 오디오 (SoundEffect, Song)
- `Microsoft.Xna.Framework.Content` — 콘텐츠 파이프라인 (ContentManager)
- `Microsoft.Xna.Framework.Media` — 미디어 재생 (MediaPlayer)

## 에이전트 활용 가이드
- 게임 구조 설계 → monogame-architect 에이전트 호출
- 스프라이트/렌더링 작업 → monogame-graphics 에이전트 호출
- 키보드/마우스/패드 처리 → monogame-input 에이전트 호출
- 사운드 이펙트/BGM → monogame-audio 에이전트 호출
- .mgcb 에셋 관리 → monogame-content 에이전트 호출
- 충돌/물리 구현 → monogame-physics 에이전트 호출
- 씬/상태 관리 → monogame-gamestate 에이전트 호출
- 성능/버그 분석 → monogame-debugger 에이전트 호출