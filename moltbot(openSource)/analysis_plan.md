 # Moltbot 코드베이스 분석 및 학습 계획

## 1. 코드베이스 전체 분류 (High-Level Classification)

Moltbot 프로젝트는 WhatsApp 게이트웨이 및 멀티 플랫폼 봇 에이전트 기능을 제공하는 복합적인 시스템으로 보입니다. 전체 구조는 다음과 같이 크게 분류할 수 있습니다.

### 1.1 Root & Configuration
프로젝트의 빌드, 테스트, 린팅, Docker 설정을 담당하는 파일들입니다.
- **주요 파일**: `package.json` (메타데이터, 스크립트), `tsconfig.json` (TypeScript 설정), `docker-compose.yml`, `Dockerfile` (컨테이너화)
- **스크립트**: `scripts/` 폴더 내에 다양한 유틸리티 및 빌드 스크립트 존재

### 1.2 Core Logic (`src/`)
봇의 핵심 기능이 구현된 영역입니다. `dist/`로 컴파일되어 실행됩니다.
- **Gateway**: `src/gateway` - 다양한 메신저 플랫폼과의 연결을 담당하는 것으로 추정
- **Agents**: `src/agents` - AI 에이전트 로직 (Pi Agent 관련)
- **Plugins/Extensions System**: `src/plugins`, `src/plugin-sdk` - 확장성 관리
- **Channels**: `src/channels` - 각 통신 채널의 추상화 계층
- **Infrastructure**: `src/infra` - 로깅, 데이터베이스 등 기반 시설
- **CLI & Commands**: `src/cli`, `src/commands` - 사용자 인터페이스 및 명령어 처리

### 1.3 Extensions (`extensions/`)
다양한 외부 플랫폼 및 서드파티 서비스와의 연동을 위한 모듈들입니다.
- **메신저**: `discord`, `slack`, `telegram`, `whatsapp`, `line`, `msteams`, `googlechat`, `imessage` 등
- **기능 확장**: `bluebubbles`, `copilot-proxy`, `google-gemini-cli-auth` 등

### 1.4 Native Apps (`apps/`)
모바일 및 데스크톱 환경을 위한 네이티브 클라이언트 앱들입니다.
- `android`: Android 애플리케이션
- `ios`: iOS 애플리케이션
- `macos`: macOS 애플리케이션
- `shared`: 앱 간 공유 코드

### 1.5 Shared Packages (`packages/`)
- `clawdbot`: 핵심 로직을 패키지화한 것으로 추정되는 폴더

---

## 2. 코드 해체 분석 및 학습 계획 (Deconstruction & Study Plan)

코드베이스의 규모가 크므로, 핵심 흐름부터 파악하고 세부 모듈로 확장하는 Top-Down 방식을 제안합니다.

### 2.1 1단계: 핵심 아키텍처 및 실행 흐름 파악
가장 먼저 시스템이 어떻게 시작되고, 핵심 루프가 어떻게 도는지를 이해합니다.
- **목표**: 엔트리 포인트에서 시작하여 초기화 과정 추적
- **분석 대상**:
    - `moltbot.mjs` 및 `src/index.ts` (또는 `src/cli` 진입점) 분석
    - `package.json`의 `scripts` (`dev`, `start`)가 실행하는 명령 추적
    - 주요 컴포넌트(`Gateway`, `Agent`, `Server`)의 인스턴스화 과정 분석

### 2.2 2단계: 데이터 흐름 및 이벤트 처리 (Gateway -> Agent)
외부 메시지가 들어왔을 때 어떻게 처리되는지 추적합니다.
- **목표**: 메시지 수신부터 응답까지의 라이프사이클 이해
- **분석 대상**:
    - **Gateway**: `src/gateway`가 어떻게 외부 입력을 받는지
    - **Routing**: `src/routing` 또는 `src/channels`에서 메시지가 어떻게 라우팅되는지
    - **Event Loop**: 메시지가 들어왔을 때 어떤 이벤트를 발생시키고 누가 구독하는지

### 2.3 3단계: 플러그인 및 익스텐션 시스템 분석
시스템의 확장성을 담당하는 구조를 파악합니다.
- **목표**: 새로운 플랫폼이나 기능을 추가하려면 어떻게 해야 하는지 이해
- **분석 대상**:
    - `src/plugin-sdk`: 플러그인 인터페이스 정의
    - `extensions/`: 대표적인 익스텐션 (예: `discord` 또는 `slack`) 하나를 선정하여 구현 패턴 분석
    - 플러그인 로딩 메커니즘 (`src/plugins`)

### 2.4 4단계: 에이전트 및 AI 기능 심층 분석
이 봇의 "지능"에 해당하는 부분을 봅니다.
- **목표**: AI 모델(OpenAI, Anthropic 등)과의 상호작용 방식 이해
- **분석 대상**:
    - `src/agents`: 에이전트 로직 및 상태 관리
    - `pi-agent-core` 등 외부 의존성 라이브러리의 활용 방식
    - 프롬프트 관리 및 컨텍스트 처리 방식

### 2.5 5단계: 인프라 및 유틸리티 (선택적)
- **분석 대상**:
    - `src/infra`: 데이터 저장소(DB), 로깅 설정
    - `src/config`: 설정 파일 로딩 및 관리

## 3. 학습 진행 방식 (Action Items)

각 단계별로 분석한 내용을 `study_moltbot` 폴더에 문서화합니다.
1.  **Architecture_Overview.md**: 시스템 전체 구조도 및 실행 흐름 정리
2.  **Message_Flow.md**: 메시지 처리 시퀀스 다이어그램 작성
3.  **Extension_Guide.md**: 익스텐션 구조 및 개발 가이드
4.  **Agent_Logic.md**: 에이전트 동작 원리 정리

이 계획에 따라 순차적으로 코드를 읽고, 실행(가능한 경우)하며 문서를 작성할 예정입니다.
