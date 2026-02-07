# Moltbot 컴퓨터 제어 (Computer Control) 구현 분석

Moltbot이 LLM(Large Language Model)을 통해 컴퓨터를 조종하는 방식은 크게 **터미널 제어 (Shell/Bash)**와 **브라우저 제어 (Browser/Playwright)** 두 가지 핵심 축으로 구현되어 있습니다.

이 문서는 각 영역이 어떻게 구현되었는지, 어떤 원리로 동작하며 최적화되어 있는지 분석한 내용입니다.

---

## 1. 터미널 제어 (Terminal Control)
LLM이 리눅스 쉘 명령어를 실행하고, 결과를 확인하며, 장기 실행 프로세스(Long-running Process)를 관리하는 방식입니다.

### 1.1 핵심 모듈
*   **`src/agents/bash-tools.exec.ts`**: 단발성 명령어 실행 (`exec` 도구).
*   **`src/agents/bash-tools.process.ts`**: 지속적인 프로세스 관리 (`process` 도구).
*   **`src/agents/bash-process-registry.ts`**: 실행 중인 프로세스 세션 상태 관리.
*   **`src/agents/pty-keys.ts`**: 키 입력(Ctrl+C 등)을 터미널에 전달하기 위한 인코딩 로직.

### 1.2 구현 원리
1.  **가상 터미널 (PTY) 활용**: 단순한 `child_process.exec`가 아니라, PTY(Pseudo Terminal)와 유사한 환경을 에뮬레이션하여 상호작용 가능한 쉘 세션을 생성합니다.
2.  **세션 관리 (Session Registry)**: `bash-process-registry.ts`에서 실행 중인 프로세스를 `sessionId`로 관리합니다. LLM은 이 ID를 통해 실행 중인 프로세스에 다시 접근할 수 있습니다.
3.  **도구(Tool) 인터페이스**: LLM에게 다음과 같은 함수들을 제공합니다.
    *   `process.list`: 실행 중인 프로세스 목록 확인.
    *   `process.poll`: 특정 세션의 최신 출력(Logs) 확인.
    *   `process.write`: `stdin`에 텍스트 입력.
    *   `process.send-keys`: 특수 키(Hex 코드, Ctrl+C 등) 전송.
    *   `process.kill`: 프로세스 종료.

### 1.3 최적화 및 특징
*   **백그라운드 실행**: 명령어가 오래 걸리면 즉시 응답을 기다리지 않고 `background` 상태로 전환하여 LLM이 다른 작업을 할 수 있게 합니다. (`poll`로 나중에 확인)
*   **입력 인코딩**: `pty-keys.ts`를 통해 ANSI 이스케이프 시퀀스나 제어 문자를 정확하게 처리하여 `vim` 같은 TUI 앱 조작 가능성도 열어두었습니다.

---

## 2. 브라우저 제어 (Browser Control)
LLM이 웹 브라우저를 띄우고, 클릭/입력/스크롤 등의 동작을 수행하는 방식입니다. Microsoft의 **Playwright**를 기반으로 구현되었습니다.

### 2.1 핵심 모듈
*   **`src/browser/pw-session.ts`**: Playwright 브라우저 세션 및 상태(Console, Network, Error) 관리.
*   **`src/browser/pw-tools-core.interactions.ts`**: 클릭, 타이핑 등 실제 동작 구현.
*   **`src/agents/tools/browser-tool.js`**: LLM에게 노출되는 브라우저 도구 정의.

### 2.2 구현 원리: Role Refs & Persistent CDP
이 구현의 가장 독창적인 부분은 **"Role Refs"** 시스템과 **지속적 CDP 연결**입니다.

1.  **지속적 연결 (Persistent Connection)**:
    *   `chromium.connectOverCDP`를 사용하여 이미 실행 중인 크롬 인스턴스에 접속합니다.
    *   이를 통해 브라우저를 매번 새로 띄우지 않고 상태를 유지합니다.

2.  **Role Refs (역할 참조 시스템) - 최적화 핵심**:
    *   **문제**: LLM에게 복잡한 HTML DOM 전체를 주면 토큰을 많이 소모하고 혼란을 줍니다.
    *   **해결**:
        *   페이지의 주요 요소(버튼, 입력창 등)를 분석하여 고유한 **Ref ID** (예: `ref=12`, `@submit`)를 부여합니다.
        *   `snapshot` 기능을 통해 LLM에게는 "Ref ID"와 "요소 설명(Role/Name)"만 요약해서 보여줍니다.
        *   LLM이 `click(ref="12")`와 같이 명령하면, 내부적으로 `refLocator` 함수가 이를 실제 Playwright Locator로 변환하여 실행합니다.
    *   **효과**: 토큰 사용량을 획기적으로 줄이고, LLM이 UI 요소를 더 정확하게 지칭할 수 있습니다.

3.  **Vision 기반 액션**:
    *   `screenshotWithLabelsViaPlaywright`: 화면 캡쳐 시, 상호작용 가능한 요소 위에 **시각적 라벨(번호표)**을 렌더링하고 스크린샷을 찍습니다.
    *   시각적 모델(GPT-4o, Claude 3.5 Sonnet 등)은 이 스크린샷의 번호를 보고 "3번 버튼 클릭해줘"라고 판단할 수 있습니다.

### 2.3 주요 함수 기능
*   `clickViaPlaywright`: Ref ID를 사용하여 요소 클릭.
*   `typeViaPlaywright`: 텍스트 입력.
*   `highlightViaPlaywright`: LLM이 주목해야 할 요소를 시각적으로 강조.
*   `scrollIntoViewViaPlaywright`: 보이지 않는 요소를 화면 안으로 스크롤.

---

## 3. 요약: LLM과 컴퓨터의 연결 고리

Moltbot은 LLM이 컴퓨터를 "블랙박스"가 아닌 **"상호작용 가능한 API 대상"**으로 인식하게 만듭니다.

1.  **추상화**: 복잡한 터미널/브라우저 동작을 `sessionId`나 `refId` 같은 핸들(Handle)로 추상화했습니다.
2.  **상태 유지**: 단발성 실행이 아니라, 세션 레지스트리와 브라우저 컨텍스트 맵(`WeakMap`)을 통해 지속적인 작업 흐름(Workflow)을 지원합니다.
3.  **토큰 효율성**: DOM 트리나 전체 로그를 덤프하는 대신, 필요한 정보(Ref, 요약 로그)만 정제하여 LLM에게 전달함으로써 비용과 정확도를 최적화했습니다.
