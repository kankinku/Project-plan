아래는 **이번 프로젝트를 바로 구현·운영할 수 있도록 정리한 “에이전트 + 스킬 통합 문서”**입니다.  
불필요한 설명은 제거하고, **① 어떤 에이전트가 ② 어떤 스킬을 ③ 어떻게 장착해서 ④ 어떤 책임을 지는지**만 남겼습니다.  
(_skills.sh 기반 스킬 + 자체 정의 스킬을 모두 포함_)

---

# 단일 시나리오 수렴형 분석 시스템

## Agent + Skill 통합 설계 문서 (Final)

---

## 0. 기본 원칙 (불변 규칙)

- **에이전트는 판단하지 않는다** → Judge만 판단
    
- **스킬은 능력이고, 에이전트는 책임 주체**
    
- **스킬은 재사용 가능, 에이전트는 교체 가능**
    
- 모든 에이전트는:
    
    - 입력 스키마 고정
        
    - 출력 스키마 고정
        
    - 권한 외 행동 금지
        

---

## 1. Orchestrator (Conductor Agent)

### 역할

- 전체 실행 흐름 제어
    
- 에이전트 호출 순서 관리
    
- 검색 루프 재실행 여부 결정
    
- **최종 시나리오가 1개 나올 때까지 반복**
    

### 장착 스킬

- ❌ 외부 스킬 불필요
    
- ✅ 내부 오케스트레이션 로직만 필요
    

### 스킬 주입 방식

```text
Conductor
- uses: []
```

---

## 2. Claim & Question Decomposer Agent

### 역할

- 입력 텍스트를 **주장(Claim)**과 **질문 유형**으로 분해
    
- 판단/해석 금지 (구조화만)
    

### 장착 스킬

- `prompt-engineering-patterns` (skills.sh)
    
- (자체) `issue-decomposition`
    

### 스킬 주입 방식

```text
DecomposerAgent
- uses:
  - prompt-engineering-patterns
  - issue-decomposition
```

### 책임 출력

- claim list
    
- question_types
    
- 핵심 엔티티/관계 태그
    

---

## 3. Reasoning Selector Agent

### 역할

- 질문 유형을 기반으로 **사용할 추론 에이전트 결정**
    
- 시나리오 개수 목표 설정(3~5)
    

### 장착 스킬

- (자체) `reasoning-selector`
    

### 스킬 주입 방식

```text
ReasoningSelectorAgent
- uses:
  - reasoning-selector
```

---

## 4. Scenario Generator Agents (병렬)

### 공통 규칙

- **결론 아님 → “후보 시나리오” 생성**
    
- assumptions / pivotal assumptions 필수
    
- 자기 약점 공개 필수
    

---

### 4-1. Abductive Scenario Agent

**역할**: 인과적 설명 시나리오 생성

```text
AbductiveAgent
- uses:
  - abductive-generator
```

---

### 4-2. Inductive Scenario Agent

**역할**: 패턴/통계 기반 시나리오 생성

```text
InductiveAgent
- uses:
  - inductive-generator
```

---

### 4-3. Analogical Scenario Agent

**역할**: 과거 유사 사례 기반 시나리오 생성

```text
AnalogicalAgent
- uses:
  - analogical-generator
```

---

### (선택) Bayesian / ToT / GoT Agent

```text
BayesianAgent
- uses:
  - bayesian-reasoning

ToTAgent
- uses:
  - tree-of-thought

GoTAgent
- uses:
  - graph-of-thought
```

※ 필요 없으면 **아예 장착하지 않음**

---

## 5. Pivotal Assumption Finder Agent

### 역할

- 각 시나리오의 **결정적 전제(pivotal assumption)** 추출
    
- 필요한 증거 타입(Fact / Stats / Mechanism) 지정
    
- 정보 공백 탐지
    

### 장착 스킬

- (자체) `pivotal-assumption-extractor`
    

```text
PivotalFinderAgent
- uses:
  - pivotal-assumption-extractor
```

---

## 6. Evidence Planner Agent

### 역할

- 어떤 전제를 위해
    
- 무엇을
    
- 어디서
    
- 어떻게 찾을지 계획 수립
    

### 장착 스킬

- (자체) `evidence-planner`
    

```text
EvidencePlannerAgent
- uses:
  - evidence-planner
```

---

## 7. Evidence Retrieval Agent

### 역할

- 실제 검색 수행
    
- **Evidence Card만 생성**
    
- 판단/해석 금지
    

### 장착 스킬

- `agent-browser` (skills.sh)
    
- (선택) `evidence-retrieval-strategist`
    

```text
EvidenceRetrievalAgent
- uses:
  - agent-browser
  - evidence-retrieval-strategist
```

---

## 8. Cross-Critique Agent

### 역할

- 후보 시나리오 간 상호 비판
    
- 취약점 라벨링
    
- 새 시나리오 생성 금지
    

### 장착 스킬

- (자체) `cross-critique`
    

```text
CrossCritiqueAgent
- uses:
  - cross-critique
```

---

## 9. 3-Layer Verifier Agent

### 역할

- **Layer 1**: 논리 검증
    
- **Layer 2**: 전제 신뢰성
    
- **Layer 3**: 현실 일관성
    

### 장착 스킬

- `verification-before-completion` (skills.sh)
    
- (자체) `three-layer-verifier`
    

```text
VerifierAgent
- uses:
  - verification-before-completion
  - three-layer-verifier
```

---

## 10. Judge / Convergence Agent (유일한 판결자)

### 역할

- 모든 후보 스코어링
    
- 타이브레이커 적용
    
- **단 하나의 시나리오 선택**
    
- 필요 시 검색 루프 재요청
    

### 장착 스킬

- `skill-judge` (skills.sh)
    
- (자체) `scenario-judge`
    

```text
JudgeAgent
- uses:
  - skill-judge
  - scenario-judge
```

### 불변 규칙

- 새 시나리오 창작 금지
    
- 후보 중 **선택만 가능**
    

---

## 11. Report Agent

### 역할

- **기각 중심 보고서** 작성
    
- Final Scenario 1개 명시
    
- 반증 체크리스트 포함
    

### 장착 스킬

- `writing-clearly-and-concisely` (skills.sh)
    
- (자체) `report-generator`
    

```text
ReportAgent
- uses:
  - writing-clearly-and-concisely
  - report-generator
```

---

## 12. Failure Ontology Manager (백그라운드)

### 역할

- 실패 유형 축적
    
- 다음 실행 시 검증 강화 신호 제공
    

### 장착 스킬

- (자체) `failure-ontology-manager`
    

```text
FailureOntologyAgent
- uses:
  - failure-ontology-manager
```

---

# 전체 구조 한눈에 요약

```text
Input
 → Decomposer
 → ReasoningSelector
 → ScenarioGenerators (병렬)
 → PivotalFinder
 → EvidencePlanner
 → EvidenceRetrieval (loop 가능)
 → CrossCritique
 → Verifier
 → Judge (단일 시나리오 확정)
 → Report
 → FailureOntology Update
```

---

## 최종 정리

- **skills.sh 스킬은 “보조 수단”**
    
- **핵심 지능은 자체 스킬로 설계**
    
- 에이전트 = 책임
    
- 스킬 = 능력
    
- Judge만 판단
    

이 문서는 그대로:

- 설계 문서
    
- README
    
- GPTs System Prompt 분할 설계
    
- 멀티에이전트 오케스트레이션 기준
    

으로 사용 가능합니다.

---

다음 단계로 가장 자연스러운 건:  
1️⃣ **각 agent별 System Prompt 실제 문안**  
2️⃣ **각 skill의 skill.md 템플릿 (skills.sh 호환)**  
3️⃣ **Conductor 상태머신 pseudo-code**

원하는 걸 번호로 말해줘.  
그다음 단계는 “설계 → 실제 실행”으로 바로 넘어갑니다.