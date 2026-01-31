# Global Constitution 기반 멀티에이전트 프로젝트 설계도 (정교본)

작성 목적: 지금까지 합의된 전체 내용을 **처음부터 끝까지** 일관된 아키텍처로 정리하고, 실제 운영/구현이 가능한 수준의 **정교한 프로젝트 설계도**로 문서화한다.

---

## 0. 요약(Executive Summary)

본 프로젝트는 다음을 목표로 하는 **멀티에이전트 운영 체계**를 구축한다.

- **Global Agents Layer**를 “헌법(Constitution) + DB(Registry)”로 고정하여, 모든 프로젝트에서 공통으로 적용되는 규칙·정의·스킬 레지스트리를 제공한다.
    
- **Workspace Layer**에서 프로젝트별 목표/범위/성공기준/라우팅 정책을 오버레이하여, Global을 수정하지 않고도 프로젝트를 구성·운영한다.
    
- 작업 흐름은 **탑다운 실행 프로토콜(Stage 1~5)**을 강제한다.
    
- 특히, 하위 에이전트 지시 후 **문서화 및 검증(Instruction/Verification Lock)**이 없으면 작업이 “존재하지 않는 것으로 간주”되도록 하여, 작업 누락/암묵 완료/순서 붕괴를 구조적으로 방지한다.
    

---

## 1. 문제 정의 및 설계 원칙

### 1.1 해결하려는 핵심 문제

1. **스킬이 늘어날수록 관리/발견성 저하**
    

- 단일 폴더/평면 구조에서는 스킬이 100개+가 되면 탐색/관리 비용이 폭증한다.
    

2. **분류(Taxonomy) 애매함으로 인한 병목**
    

- 예: “API 보안 테스트”가 Web/Security/Testing 어디에 속하는지 애매해 배치·탐색 혼선 발생.
    

3. **경로(Path) 하드코딩으로 인한 마이그레이션 비용**
    

- `skills_index.json`에 경로가 하드코딩되어 있으면 폴더 구조 변경 시 광범위 수정이 필요.
    

4. **에이전트 실행이 문서화/검증 없이 종료되는 실패 모드**
    

- 상위 에이전트가 하위 에이전트에 지시해도, 완료 확인이 문서화되지 않으면 작업이 누락되거나 암묵적으로 종료될 수 있음.
    

### 1.2 핵심 설계 원칙

- **Single Source of Truth(단일 진실)**: 스킬/에이전트 정의는 Global Registry만이 진실.
    
- **ID 기반 참조**: 경로가 아닌 `skill_id`를 기준으로 연결(폴더 구조 변경과 무관).
    
- **Top-down 강제**: 목표→분해→명세→수단 제한→실행/검증. 순서 위반 불가.
    
- **Instruction/Verification Lock**: 문서로 검증되지 않은 작업은 존재하지 않는 것으로 간주.
    
- **Global 불변, Workspace 가변**: 프로젝트 변경은 Workspace에서만.
    

---

## 2. 전체 아키텍처 개요

### 2.1 계층 구조

- **Global Agents Layer**: 헌법 + DB(레지스트리) + 스킬 저장소
    
- **Workspace Layer**: 프로젝트별 설정/정책/작업 산출물
    
- **Execution Layer**: 하위 에이전트 실행 및 산출물 생성
    

### 2.2 컨텍스트 병합 우선순위(Override Order)

1. Global Constitution (최상위, override 불가)
    
2. Global Registry (skills_index, agent_catalog)
    
3. Workspace Project Config (프로젝트 오버레이)
    
4. Task Spec (현재 작업 단위)
    

이 순서를 벗어나면 시스템이 “프로젝트별 변덕” 또는 “에이전트 임의 판단”으로 붕괴한다.

---

## 3. Global Agents Layer 설계 (헌법 + DB)

### 3.1 역할 정의

Global Agents는 두 가지 기능을 수행한다.

1. **헌법(Constitution) 역할**
    

- 모든 에이전트가 따라야 할 불변 규칙(안전/품질/근거/작업 순서/실패 처리)
    

2. **데이터베이스(DB) 역할**
    

- 스킬 레지스트리(검색/라우팅/resolve)
    
- 에이전트 타입 카탈로그(역할 정의)
    

### 3.2 Global 폴더 구조(권장)

```
/global_agents
  /constitution
    constitution.md
    quality_gates.md
    evidence_policy.md
    task_protocol.md
    instruction_verification_lock.md

  /registry
    skills_index.json
    agent_catalog.json
    routing_primitives.yaml

  /skills
    <tier1>/<tier2>/<skill_folder>/skill.md
```

### 3.3 Global Constitution 구성

- **constitution.md**: 최상위 원칙(우선순위/금지/권한)
    
- **quality_gates.md**: 정의된 완료 조건(DOD)·품질 게이트
    
- **evidence_policy.md**: 근거/출처/검증 정책
    
- **task_protocol.md**: 탑다운 실행 프로토콜(Stage 1~5) 규정
    
- **instruction_verification_lock.md**: Instruction–Verification Lock 규정(아래 7장)
    

### 3.4 Global Registry 구성

#### 3.4.1 skills_index.json (필수)

- `skill_id`: 불변 식별자
    
- `path`: 실제 폴더 위치(변경 가능)
    
- `name`, `description`
    
- `use_when`: 라우팅을 위한 사용 맥락
    
- `tags`, `aliases`: 애매한 분류 해결 및 검색 확장
    
- `risk_level`(선택): 위험한 스킬의 게이트
    

> 핵심: 모든 참조는 `skill_id`로만. `path`는 registry가 resolve.

#### 3.4.2 agent_catalog.json (필수)

- 에이전트 “타입” 정의(프로젝트 비종속)
    
- 각 타입의 책임/권한/기본 스킬셋(Core)
    

예: orchestrator, router, researcher, implementer, reviewer, validator

#### 3.4.3 routing_primitives.yaml (선택)

- 라우팅에 사용되는 전역 규칙 조각(예: 태그 기반 매칭, 금지 스킬 목록)
    
- Workspace가 이를 조합해 프로젝트 정책 구성
    

---

## 4. Skills 분류 체계(최대 3레벨) 설계

### 4.1 폴더 구조 규격

- Root: `/global_agents/skills/`
    
- 3레벨 제한:
    
    - Tier 1(대분류): 기능적 영역(Functional Domain)
        
    - Tier 2(소분류): 구체 기술/대상/목적(Subdomain)
        
    - Tier 3: 스킬 폴더(Skill)
        

예:  
`skills/security/identity/active-directory-attacks`

### 4.2 분류 원칙

- **Use case(업무 의도) 우선**
    
- 한 스킬은 **하나의 canonical home**만 갖는다.
    
- 애매함은 폴더를 늘리는 대신 `tags/aliases`로 해결
    

### 4.3 Taxonomy Problem 해결 규칙(결정 우선순위)

1. 주요 사용 의도(Use when)
    
2. 산출물 형태(리포트/코드/설정/테스트 결과)
    
3. 실패/리스크가 어디에서 발생하는가(보안이면 security 우선)
    
4. 플랫폼 종속이면 platform 하위로 이동
    

### 4.4 경로 관리 문제(마이그레이션) 해결

- 에이전트/프로젝트는 경로를 알 필요가 없다.
    
- 폴더 이동 시:
    
    - skills_index.json의 `path`만 업데이트
        
    - 또는 index 자동 재생성으로 동기화
        

---

## 5. Workspace Layer 설계 (프로젝트 오버레이)

### 5.1 Workspace의 책임

Workspace는 “프로젝트마다 달라지는 요소”만 보유한다.

- 프로젝트 목표/범위/성공기준
    
- 프로젝트 도메인 용어집/온톨로지
    
- 스킬 라우팅 정책(프로젝트별 우선순위)
    
- 에이전트 인스턴스 설정(어떤 타입을 어떻게 쓸지)
    
- 계획 산출물(WBS/Task Spec/리포트)
    

### 5.2 Workspace 폴더 구조(권장)

```
/workspace
  /project
    project_config.yaml
    project_charter.md
    glossary.md
    ontology.yaml

  /agents
    workspace_orchestrator.md
    workspace_router.md
    workspace_reviewer.md
    workspace_validator.md

  /plans
    plan.json
    wbs.json
    /tasks
    stage_execution_log.md

  /reports
    /agent_completion
    /reviews
    /validation
```

### 5.3 project_config.yaml (핵심 오버레이 파일)

필수 섹션:

- project: objective, non_goals, acceptance, constraints
    
- routing: rules(태그/키워드 기반 → required/recommended skill_id)
    
- agents: instances(타입 + 프로젝트 추가 스킬)
    
- evidence_policy(프로젝트별 허용 출처/검증 강도)
    

> Workspace는 Global을 덮어쓰는 게 아니라 **선택/조합/우선순위**만 제공한다.

---

## 6. 탑다운 실행 프로토콜(Top-down Execution Protocol)

탑다운 실행 프로토콜은 “권장 흐름”이 아니라 **실행 조건**이다.

### Stage 1 — Project Charter 수립

- 목표(Objective), 비목표(Non-goals), 범위(Scope)
    
- 제약(Constraints), 성공 기준(Acceptance Criteria)
    

**게이트**: Charter 승인 전 구현/조사 실행 금지

### Stage 2 — WBS 생성(3~4레벨)

- 워크스트림 → 구성요소 → 구현 단위 → 티켓(선택)
    
- 의존성(DAG) 명시
    

**게이트**: 모든 실행은 WBS 상 위치를 가져야 함

### Stage 3 — Task Spec 표준화

모든 작업은 Task Spec으로 정의되어야 한다.

- 목적
    
- 입력/출력
    
- acceptance
    
- 리스크
    
- 의존성
    

**게이트**: Task Spec 없는 작업 실행 금지

### Stage 4 — Skill Routing

- 각 Task에 필수/권장 스킬을 `skill_id`로 부착
    
- 라우팅: Global Registry + Workspace 정책 기반
    

**게이트**: 스킬 지정 없는 태스크 실행 금지

### Stage 5 — Execution → Review → Validation

- 실행(implement/research)
    
- 리뷰(reviewer)
    
- 검증(validator: acceptance 충족 확인)
    

**게이트**: acceptance 통과 전 완료 선언 금지

### 실패 처리(롤백)

- 실패는 “구현” 문제가 아니라 “의사결정 단계” 문제로 간주
    
- 결함이 발생한 상위 단계로 롤백 후 문서 수정
    
- 임시 패치/우회 금지
    

---

## 7. Instruction–Verification Lock (GC-IV-001)

### 7.1 문제 배경

하위 에이전트에 작업을 지시해도, 완료 확인이 문서화되지 않으면:

- 작업이 조용히 사라짐
    
- 완료 착각
    
- 순서/의존성 무시
    

이를 구조적으로 차단하기 위해 Global Constitution 규칙으로 승격한다.

### 7.2 규칙 정의

> **Instruction 문서로 발행되고 Completion Record로 검증되지 않은 작업은,  
> 시스템 상 “실행되지 않은 작업”으로 간주한다.**

### 7.3 필수 문서 2종

#### (1) Stage Execution Log (Workspace 필수)

- 단계별로 “누구에게 무엇을 지시했는지” 기록
    
- Instruction 단위로 추적
    

필드(최소):

- Instruction ID
    
- Issuer(Orchestrator)
    
- Target Agent
    
- Task Reference
    
- Required Deliverables
    
- Acceptance Reference
    
- Status: In Progress / Completed(Self) / Verified / Rejected
    
- Completion Record 링크
    

#### (2) Agent Completion Record (Workspace 필수)

- 하위 에이전트가 완료를 주장하는 증빙 문서
    

필드(최소):

- Instruction ID
    
- Agent
    
- 수행 요약
    
- 산출물 목록(링크)
    
- Acceptance Checklist
    
- 리스크/오픈 이슈
    
- Self-declaration: COMPLETED
    

### 7.4 Stage 종료 조건(필요충분)

> **해당 Stage Execution Log에 존재하는 모든 Instruction이 Verified일 것**

- 하나라도 미검증이면 Stage 종료 불가
    
- Stage 미종료이면 다음 Stage 진입 불가
    

---

## 8. 에이전트 구성(타입 vs 인스턴스)

### 8.1 Global: Agent Type Catalog

Global은 타입 정의만 제공한다.

- orchestrator: 지시/추적/검증/차단(실행자 아님)
    
- router: 태스크→스킬 매핑
    
- researcher: 근거 수집/정리(RAG 포함)
    
- implementer: 구현/통합
    
- reviewer: 구조/논리/품질 검토
    
- validator: acceptance 검증/현실성 체크
    

### 8.2 Workspace: Agent Instances

프로젝트는 필요한 타입을 인스턴스화하고, 프로젝트별 추가 스킬/정책을 부여한다.

필수 인스턴스(권장 최소 4개):

- workspace_orchestrator
    
- workspace_router
    
- workspace_reviewer
    
- workspace_validator
    

필요 시 추가:

- researcher
    
- implementer
    

---

## 9. Orchestrator 강제 로직(Instruction/Verification 삽입)

Orchestrator는 “일하는 에이전트”가 아니라 “일이 사라지지 않게 잠그는 에이전트”다.

### 9.1 ORCH 강제 규칙

- **ORCH-001 Instruction First**: 하위 작업 요청 전 Instruction ID 생성 + Execution Log 기록
    
- **ORCH-002 No Silent Completion**: Completion Record 없는 응답은 무시하고 제출 요구
    
- **ORCH-003 Verification Gate**: reviewer/validator 확인 후에만 Verified로 전환
    
- **ORCH-004 Stage Exit Lock**: 미검증 Instruction 존재 시 Stage 종료/전이 차단
    

### 9.2 상태 모델(권장)

- In Progress: 지시됨
    
- Completed(Self): 하위 에이전트 완료 선언(Completion Record 존재)
    
- Verified: reviewer/validator 확인 완료
    
- Rejected: 기준 미달(롤백/재작업)
    

---

## 10. 운영 산출물 및 데이터 흐름

### 10.1 핵심 산출물

- project_charter.md
    
- wbs.json
    
- tasks/*.json (Task Specs)
    
- stage_execution_log.md
    
- agent_completion/*.md
    
- reviews/*.md
    
- validation/*.md
    
- plan.json (상위 요약/마일스톤)
    

### 10.2 전형적인 실행 시퀀스

1. Stage 1 문서 생성/고정
    
2. Stage 2 WBS 생성
    
3. Stage 3 Task Specs 생성
    
4. Stage 4 Skill Routing
    
5. Stage 5 Execution→Review→Validation
    
6. 실패 시 해당 Stage로 롤백하고 문서 갱신
    

---

## 11. 리스크 및 대응

### 11.1 Taxonomy 병목

- 대응: canonical home + tags/aliases, 폴더 확장 최소화
    

### 11.2 문서 오버헤드

- 대응: 템플릿/자동 생성, 최소 필드 강제, stage gate로 가성비 확보
    

### 11.3 라우팅 누락(스킬 선택 실패)

- 대응: use_when 강화, Router 규칙 우선 + 불확실 시 선택지 2~3개 제안
    

### 11.4 “응답은 왔는데 문서 없음” 반복

- 대응: ORCH-002로 응답 무시 + Completion Record 제출 요구를 자동화
    

---

## 12. 구현 로드맵(Top-down)

### Milestone 0 — Global 헌법+DB 고정

- Constitution 문서 5종 생성
    
- skills_index.json 확정(필수 필드 포함)
    
- agent_catalog.json 확정
    

### Milestone 1 — Workspace 기본 골격

- project_config.yaml
    
- project_charter.md
    
- stage_execution_log.md 템플릿
    
- agent_completion 템플릿
    

### Milestone 2 — Orchestrator/Router/Reviewer/Validator 인스턴스

- ORCH 강제 로직 포함 프롬프트 적용
    
- Router 규칙 적용
    

### Milestone 3 — 1회전 시뮬레이션(End-to-end)

- Stage 1~5를 실제로 수행
    
- 검증 실패 케이스를 일부러 발생시켜 롤백이 작동하는지 확인
    

### Milestone 4 — 자동화 강화

- skills_index 자동 생성/검증
    
- Stage Log/Completion Record 자동 템플릿 생성
    
- 상태 전이 검사(미검증 instruction 존재 시 전이 차단)
    

---

## 13. 완료 정의(Definition of Done)

본 시스템은 아래가 충족되면 “운영 가능” 상태로 간주한다.

- 모든 프로젝트가 Global을 수정하지 않고 Workspace만으로 구동 가능
    
- Stage 1~5 게이트가 실제로 위반 불가하게 동작
    
- Instruction/Verification Lock이 실제로 작업 누락을 차단
    
- skill_id 기반 resolve로 폴더 구조 변경이 시스템을 깨지 않음
    

---

## 14. 부록: 템플릿(요약)

### A) Stage Execution Log – 최소 템플릿(요약)

- Stage
    
- Instruction ID
    
- Target Agent
    
- Task Ref
    
- Deliverables
    
- Acceptance
    
- Status
    
- Completion Record
    

### B) Agent Completion Record – 최소 템플릿(요약)

- Instruction ID
    
- Agent
    
- Delivered Artifacts
    
- Acceptance Checklist
    
- Risks/Open Questions
    
- Declaration
    

---

## 결론

이 설계의 본질은 “에이전트가 똑똑해지기를 기대하는 것”이 아니라,

- **Global 헌법으로 규칙을 고정하고**
    
- **Workspace 오버레이로 프로젝트 변수를 흡수하며**
    
- **탑다운 게이트와 Instruction/Verification Lock으로 실패 모드를 구조적으로 제거**
    

하여, 에이전트가 덜 완벽해도 프로젝트가 흔들리지 않게 만드는 것이다.