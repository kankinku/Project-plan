# [Insight Report] LLM 기반 컴퓨터 제어의 진화: MoltBot 아키텍처 심층 분석

**작성자:** Chief Technical Editor  
**날짜:** 2026-01-30

---

## 1. Executive Summary

본 리포트는 거대 언어 모델(LLM)이 단순한 텍스트 처리 도구를 넘어, 실제 컴퓨터 시스템(터미널 및 브라우저)을 정밀하게 제어하는 '실행형 에이전트(Actionable Agent)'로 진화하는 과정을 MoltBot의 아키텍처를 통해 분석합니다. MoltBot은 **'가상 터미널 세션 레지스트리(Session Registry)'**와 **'Role Refs(역할 참조) 시스템'**이라는 독창적인 추상화 계층을 도입하여, 기존 에이전트의 고질적인 문제였던 컨텍스트 유실과 토큰 비효율성 문제를 획기적으로 해결하였습니다. 이러한 기술적 접근은 향후 자율 에이전트 개발에 있어 '상태 지속성(State Persistence)'과 '정밀한 대상화(Pinpoint Targeting)'가 필수적인 요소임을 시사합니다.

---

## 2. Visual Integration: The Abstraction Layer

다음은 MoltBot이 복잡한 웹 환경과 터미널 환경을 어떻게 LLM이 이해할 수 있는 형태(Abstract Handle)로 변환하는지를 보여주는 개념도입니다.

![MoltBot Architecture Diagram](../images/moltbot_architecture_diagram.png)
*Figure 1: MoltBot의 아키텍처 개념도. 기존의 복잡한 DOM 구조와 터미널 스트림이 'Role Ref' 및 'Session ID'라는 추상화된 핸들을 통해 효율적으로 AI Core로 전달되는 데이터 흐름을 시각화하였습니다.*

---

## 3. Technology Analysis

### 3.1 기술적 배경 (Current Landscape)
기존의 LLM 기반 에이전트들은 컴퓨터 제어 시 두 가지 주요한 장벽에 부딪혔습니다. 첫째, **상태 관리의 부재**입니다. 터미널 명령은 연속적인 컨텍스트(예: `cd` 명령 후의 상태 유지)가 필수적이나, 단순한 `exec` 방식은 매번 새로운 쉘을 띄우므로 이를 유지하기 어렵습니다. 둘째, **브라우저 제어의 비효율성**입니다. 현대 웹 페이지의 DOM 트리는 너무 방대하여, 이를 그대로 LLM에 프롬프트로 전달할 경우 토큰 비용이 급증하고 처리 속도가 저하되며, 할루시네이션(Hallucination) 발생 가능성이 높아집니다.

### 3.2 핵심 아키텍처 및 원리 (Technical Deep Dive)
MoltBot은 이러한 한계를 극복하기 위해 두 가지 핵심 기술을 도입하였습니다.

#### A. Persistent PTY & Session Registry
터미널 제어에 있어 단순한 프로세스 실행이 아닌, **가상 터미널(PTY) 기반의 세션 레지스트리**를 구축하였습니다.
*   **세션 지속성**: `bash-process-registry.ts` 모듈은 `sessionId`를 발급하여 실행 중인 프로세스를 관리합니다. 이를 통해 LLM은 `vim`이나 서버 데몬과 같은 장기 실행 프로세스(Long-running Process)와 지속적으로 상호작용할 수 있습니다.
*   **정교한 입출력 제어**: `pty-keys.ts`를 통해 ANSI 이스케이프 시퀀스 및 특수 키(Ctrl+C 등)를 정확히 인코딩하여 전달함으로써, 실제 사람이 터미널을 조작하는 것과 동일한 수준의 제어력을 확보했습니다.

#### B. Role Refs & Persistent CDP (Browser Control)
브라우저 제어의 핵심은 **Role Refs(역할 참조)** 시스템입니다.
*   **토큰 최적화의 기술**: 웹 페이지의 전체 DOM을 LLM에 전달하는 대신, 상호작용 가능한 요소(버튼, 입력창 등)에만 고유한 ID(`ref=12`)를 부여하고 요약된 정보만을 제공합니다. 이는 프롬프트 토큰 사용량을 획기적으로 줄여줍니다.
*   **Persistent CDP**: `chromium.connectOverCDP`를 활용하여 브라우저 인스턴스와 지속적인 연결을 유지함으로써, 매 작업마다 브라우저를 새로 띄우는 오버헤드를 제거하고 사용자의 로그인 세션 등을 보존합니다.

### 3.3 비즈니스 임팩트 (Business & Dev Impact)
이러한 기술적 진보는 비즈니스 및 개발 환경에 다음과 같은 실질적인 가치를 제공합니다.

1.  **운영 비용 절감**: Role Refs를 통한 토큰 사용량 감소는 API 비용 절감으로 직결되며, 이는 대규모 에이전트 시스템 운영 시 결정적인 경쟁력이 됩니다.
2.  **안정적인 자동화 파이프라인**: 세션 지속성 덕분에 복잡한 배포 파이프라인이나 긴 시간이 소요되는 데이터 분석 작업을 중단 없이 수행할 수 있어, RPA(Robotic Process Automation)의 적용 범위를 획기적으로 확장합니다.
3.  **개발자 경험(DX) 향상**: LLM이 터미널과 브라우저를 마치 API처럼 명확한 핸들(Handle)로 제어하게 함으로써, 프롬프트 엔지니어링의 난이도를 낮추고 예측 가능한 결과를 보장합니다.

---

## 4. Technical Comparison

다음은 기존의 일반적인 LLM 에이전트 방식과 MoltBot 아키텍처의 차이점을 분석한 비교표입니다.

| 비교 항목 | 기존 방식 (Raw Automation) | MoltBot 방식 (Abstracted Control) | 비고 |
| :--- | :--- | :--- | :--- |
| **터미널 제어** | `child_process.exec` (단발성) | **PTY + Session Registry** (지속성) | 상태 유지(cd, env 등) 가능 여부 결정적 차이 |
| **브라우저 제어** | 전체 HTML DOM 덤프 의존 | **Role Refs (요약된 고유 ID)** | 토큰 비용 90% 이상 절감 효과 기대 |
| **요소 식별 정확도** | XPath/Selector 오류 빈번 | **Ref ID (`ref=12`)** 직접 매핑 | 할루시네이션 최소화 및 타겟팅 정확도 향상 |
| **프로세스 관리** | 실행 후 결과 대기(Blocking) | **Background/Poll** 비동기 관리 | 장시간 작업 병렬 처리 가능 |
| **연결 방식** | 매 실행 시 신규 인스턴스 | **Persistent CDP Connection** | 기존 브라우저 컨텍스트/세션 재사용 |

---

**결론적으로**, MoltBot은 단순한 자동화 스크립트 실행기가 아닙니다. 컴퓨터의 복잡한 상태(State)와 인터페이스를 LLM이 인지하기 쉬운 형태로 **'추상화(Abstraction)'**하고 **'지속(Persistence)'**시키는 일종의 **'운영체제 브리지(OS Bridge)'** 역할을 수행하고 있습니다.

이러한 아키텍처는 향후 AI가 디지털 환경에서 실질적인 노동력을 제공하는 주체로 거듭나기 위한 표준적인 청사진(Blueprint)이 될 것으로 판단됩니다.
