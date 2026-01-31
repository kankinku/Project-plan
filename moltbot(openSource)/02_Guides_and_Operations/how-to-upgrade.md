# Moltbot 시스템 통합 업그레이드 가이드 (Unified Upgrade Guide)

이 문서는 Moltbot의 복잡한 시스템 아키텍처를 이해하고, 안전하게 최신 버전으로 업그레이드하기 위한 통합 가이드입니다. 기존의 단편적인 문서들을 통합하여 개발자가 전체 시스템을 조망하며 업그레이드를 수행할 수 있도록 작성되었습니다.

## 1. 시스템 개요 (System Overview)

업그레이드를 수행하기 전, Moltbot이 일반적인 봇과 달리 왜 이렇게 구성되어 있는지 이해해야 합니다.

### 1.1 핵심 아키텍처
Moltbot은 단순한 스크립트 실행기가 아닌, **OS 제어 계층(Operating System Control Layer)**을 추상화한 분산 시스템입니다.

-   **Gateway (`src/gateway`)**: 모든 외부 요청(Slack, WhatsApp 등)의 진입점이며, 메시지 라우팅을 담당합니다.
-   **Agents (`src/agents`)**: AI 모델의 두뇌 역할을 수행하며, 실제 판단과 결정이 이루어집니다.
-   **Browser Server (`src/browser`)**: Playwright 기반의 브라우저 제어 전용 서버입니다. 에이전트와 분리되어 있어 브라우저가 충돌해도 시스템 전체가 멈추지 않습니다.
-   **Infrastructure (`src/infra`)**: Docker 기반의 샌드박스 환경과 실행 승인 시스템을 관리합니다.

### 1.2 주요 차별점
-   **안전성 (Safety)**: 위험한 명령어(`rm -rf` 등)는 사용자 승인이 필요하며, 격리된 컨테이너에서 실행됩니다.
-   **지속성 (Persistence)**: **Session Registry**를 통해 터미널 세션(`cd` 이동 상태 등)을 유지하고, **Persistent CDP**를 통해 브라우저 로그인 상태를 유지합니다.

---

## 2. 업그레이드 전 필수 사전 지식 (Prerequisites)

시스템 업그레이드 시 다음 두 가지 핵심 메커니즘에 영향을 줄 수 있으므로 주의가 필요합니다.

### 2.1 Role Refs 시스템 (Browser Control)
Moltbot은 브라우저의 DOM 전체를 LLM에게 보내지 않고, **Role Refs**라는 고유 ID(`ref=e1`)로 변환하여 통신합니다.
-   **주의사항**: `src/browser/pw-role-snapshot.ts` 등의 로직이 변경되면, 기존에 학습된 프롬프트나 스냅샷 처리 방식과 호환되지 않을 수 있습니다. 업그레이드 후 브라우저 제어가 잘 안 된다면 이 부분이 핵심 원인일 가능성이 높습니다.

### 2.2 PTY Session (Terminal Control)
터미널 제어는 단순 `exec`이 아니라 `node-pty`를 사용한 가상 터미널 세션입니다.
-   **주의사항**: 시스템을 재시작하면 기존의 모든 열려있는 터미널 세션(변수 설정, 디렉토리 위치 등)은 초기화됩니다. 운영 중인 중요 작업이 있다면 업그레이드 전 반드시 종료해야 합니다.

---

## 3. 단계별 업그레이드 가이드 (Step-by-Step Upgrade)

### 3.1 1단계: Core System 업그레이드
가장 기본이 되는 소스 코드와 의존성을 업데이트합니다.

1.  **Repository Update**:
    ```bash
    git pull origin main
    ```
2.  **의존성 설치 (Dependencies)**:
    Moltbot은 `pnpm` 또는 `npm`을 사용합니다. `package-lock.json`이나 `pnpm-lock.yaml` 변경 사항을 반영합니다.
    ```bash
    npm install
    # 또는
    pnpm install
    ```
3.  **Build**:
    TypeScript 컴파일을 다시 수행합니다.
    ```bash
    npm run build
    ```

### 3.2 2단계: Browser Server 재구성
브라우저 제어 서버는 별도의 프로세스 또는 서비스로 돌 수 있습니다.
-   Playwright 브라우저 바이너리가 업데이트되었는지 확인합니다.
    ```bash
    npx playwright install chromium
    ```
-   브라우저 서버를 재시작하여 새로운 코드를 반영합니다.

### 3.3 3단계: Infrastructure & Sandbox 업데이트
Docker를 사용하는 경우, 샌드박스 이미지를 최신으로 빌드해야 합니다.
```bash
# Docker Compose를 사용하는 경우
docker-compose build --no-cache agents
docker-compose up -d
```
*주의: `src/agents/bash-tools.shared.ts` 등 샌드박스 관련 로직이 변경되었다면 반드시 컨테이너를 재생성해야 합니다.*

---

## 4. 설정 및 마이그레이션 (Configuration & Migration)

### 4.1 설정 파일 (`config/`) 점검
-   `config/moltbot.json` (또는 `.env`) 파일에 새로운 필수 설정값이 추가되었는지 `config.example.json`과 비교하십시오.
-   특히 **LLM Provider** 설정이나 **Role Ref** 관련 튜닝 파라미터가 변경되었을 수 있습니다.

### 4.2 컴포넌트 호환성 체크
-   **Skills**: `skills/` 디렉토리의 커스텀 스킬들이 새로운 Core API와 호환되는지 확인합니다.
-   **Clients**: 모바일 앱(Android/iOS)을 사용하는 경우, 서버 API 버전 변경에 따라 앱 업데이트가 필요할 수 있습니다.

---

## 5. 검증 및 문제 해결 (Verification & Troubleshooting)

업그레이드 완료 후 다음 체크리스트를 통해 정상 동작을 확인합니다.

### 5.1 기능 검증 체크리스트
-   [ ] **Gateway 연결**: 봇이 메신저(Slack 등)에서 "Hello"에 응답하는가?
-   [ ] **Terminal 제어**: "현재 폴더의 파일 목록 보여줘"(`ls -la`) 명령이 정상 실행되는가?
-   [ ] **Browser 제어**: "Google에서 날씨 검색해줘" 명령 시 브라우저가 뜨고 검색 결과가 나오는가? (Role Refs 작동 확인)
-   [ ] **상태 유지**: "A 변수에 10 저장해" -> "A 변수 값 뭐야?" 대화가 맥락을 유지하며 이어지는가?

### 5.2 주요 트러블슈팅 (FAQ)

**Q. 브라우저가 클릭을 엉뚱한 곳에 합니다.**
-   **A.** `Role Refs` 매핑 문제일 수 있습니다. 브라우저 창의 해상도가 변경되었거나, 사이트의 UI가 변경되어 OCR/접근성 트리가 요소를 못 찾을 수 있습니다. 디버그 모드를 켜서 스냅샷 로그를 확인하세요.

**Q. "Permission Denied" 오류가 계속 뜹니다.**
-   **A.** `src/infra/exec-approvals.ts` 정책이 강화되었을 수 있습니다. 로컬 테스트라면 승인 정책을 완화하거나, 승인 UI에서 명시적으로 허용해주세요.

**Q. 봇이 응답이 없거나 멈춥니다.**
-   **A.** Docker 컨테이너의 리소스 제한(메모리 부족)일 수 있습니다. 또는 OpenAI/Anthropic API 키 만료나 Rate Limit을 확인하세요 (`src/agents/auth-profiles.ts`).