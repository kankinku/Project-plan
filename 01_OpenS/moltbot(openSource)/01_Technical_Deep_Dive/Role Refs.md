MoltBot의 "Role Ref" 메커니즘 심층 분석
---
MoltBot이 LLM에게 복잡한 웹 페이지를 보여주고 제어하는 핵심 기술인 "Role Refs(역할 참조)" 시스템을 분석했습니다. 이 시스템은 **"복잡한 DOM을 토큰 효율적인 고유 ID로 추상화"**하는 것이 핵심입니다.

1. 배경: 왜 "Role Ref"가 필요한가?
LLM에게 웹 페이지 전체 HTML 소스(<body>...</body>)를 그대로 주면 다음과 같은 문제가 발생합니다.

토큰 폭탄: HTML은 태그와 속성이 너무 많아 토큰을 엄청나게 소모합니다.
할루시네이션: 중첩된 <div> 지옥 속에서 LLM은 "로그인 버튼"이 어디 있는지 헷갈려합니다.
선택의 어려움: "로그인 버튼 눌러"라고 했을 때, 정확한 CSS Selector(div.main > button.btn-primary)를 생성하기 어렵습니다.
MoltBot은 이를 해결하기 위해 **Accessibility Tree(접근성 트리)**를 기반으로 한 독자적인 ID 매핑 시스템을 사용합니다. 이것이 바로 Role Ref입니다.

2. 작동 원리 (Workflow)
Role Ref의 생명 주기는 생성(Snapshotting) → 저장(Mapping) → 실행(Execution) 3단계로 이루어집니다.

2.1 단계 1: 생성 (Snapshotting)
담당 파일: src/browser/pw-snapshot.ts, src/browser/pw-role-snapshot.ts

접근성 스냅샷 추출: Playwright의 ariaSnapshot 기능을 사용하여 시각장애인용 스크린 리더가 보는 것과 유사한 "의미 있는 요소"들만 추출합니다.
text
- banner
  - heading "MoltBot"
- button "Login"
- textbox "Username"
ID 부여: buildRoleSnapshotFromAriaSnapshot 함수가 이 트리를 순회하며 상호작용 가능한 요소(버튼, 입력창 등)에 e1, e2... 와 같은 짧은 고유 ID를 부여합니다.
LLM 프롬프트 생성: 최종적으로 LLM에게는 다음과 같이 정리된 텍스트만 보여줍니다.
text
- heading "MoltBot"
- button "Login" [ref=e1]
- textbox "Username" [ref=e2]
2.2 단계 2: 저장 (Mapping & Storage)
담당 파일: src/browser/pw-session.ts

생성된 ID(e1)가 실제로 어떤 요소를 가리키는지 기억해야 합니다. MoltBot은 이를 **WeakMap**과 **Map**을 이용해 메모리에 저장합니다.

매핑 테이블: e1 → { role: "button", name: "Login", nth: 0 }
중복 처리 (nth): 만약 화면에 "Login" 버튼이 2개라면?
e1 → { role: "button", name: "Login", nth: 0 } (첫 번째)
e3 → { role: "button", name: "Login", nth: 1 } (두 번째)
이렇게 nth 속성으로 정확하게 구분합니다.
특이점: HTTP 요청(Stateless) 간에도 이 매핑을 유지하기 위해 targetId(탭 ID)를 키(Key)로 하는 캐시(roleRefsByTarget)를 따로 관리합니다.

2.3 단계 3: 실행 (Execution)
담당 파일: src/browser/pw-tools-core.interactions.ts

LLM이 **"click e1"**이라는 명령을 내리면 다음 과정이 일어납니다.

현재 MoltBot의 구현에서는 별도의 특수 목적(Fine-tuned) 모델이나 경량화된 로컬 모델(PaddleOCR 등)을 사용하지 않고, 메인 LLM (예: Claude 3.5 Sonnet, GPT-4o) 하나가 모든 것을 처리합니다.

이미지나 텍스트 정보를 메인 LLM에게 "잘 정리해서 떠먹여 주는(Feeding)" 방식입니다.

---

현재 방식 (Current Architecture)
인식 (Recognition):
**코드(
pw-role-snapshot.ts
)**가 브라우저의 접근성 트리(Accessibility Tree)를 읽어서 e1, e2 같은 ID를 기계적으로 붙입니다. (AI 모델 사용 X, 100% 알고리즘)
판단 (Reasoning):
이 정리된 텍스트(또는 스크린샷)를 메인 LLM에게 프롬프트로 보냅니다.
메인 LLM이 "사용자가 로그인을 원하니까, 'Login'이라고 적힌 e1 버튼을 눌러야겠군"이라고 판단합니다.

AS-IS: 무거운 메인 LLM이 화면 전체 해석 + 행동 결정까지 다 함 (비쌈, 느림).
TO-BE (제안): 가벼운 모델(PaddleOCR 등)이 화면의 글자/버튼 위치만 빠르게 찾아서 좌표를 주고, 메인 LLM은 "좌표 (100, 200) 눌러"라고 결정만 함 (빠름, 싸짐).
결론적으로, 현재 코드는 "똑똑한 LLM 하나에 전적으로 의존하는" 형태입니다.

---
문제 인식 
- 굳이 이미지를 인식하고, 컴퍼넌트를 분리하는것에 토큰을 소모해야할까?

**"PaddleOCR이나 경량 모델 사용"**은 현재의 방식에서 벗어나, "눈(Vision)" 역할을 하는 별도의 가볍고 빠른 모델을 도입하자는 아주 좋은 아이디어입니다.