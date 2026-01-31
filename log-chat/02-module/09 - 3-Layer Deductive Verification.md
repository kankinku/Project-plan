(연역 기반 형식·신뢰·현실 검증 단계)

**목적:**  
8단계까지 살아남은 후보 시나리오들에 대해  
→ **논리적으로 성립하는지**,  
→ **전제가 신뢰 가능한지**,  
→ **현실 세계와 충돌하지 않는지**를  
**연역적으로 검증**하여 _판결 자격_을 부여하거나 박탈한다.

> 핵심 포인트
> 
> - 이 단계는 **창의 0%, 판사 100%**
>     
> - 새로운 정보 생성 금지
>     
> - “그럴듯함”은 평가 대상 아님
>     

---

## 1) 입력과 출력

### 입력 (Cross-Critique Pack)

8단계 출력 전체

`{   "candidate_scenarios": [...],   "critiques": [...],   "scenario_vulnerability": [...],   "consensus_signals": [...] }`

---

### 출력 (Verification Pack)

이 출력은 **10단계(Judge / Convergence)**의 입력이 된다.

`{   "verification_results": [     {       "scenario_id": "S1",       "layer_results": {         "logical_validity": {           "status": "pass|fail|conditional",           "issues": ["non_sequitur", "circular_reasoning"]         },         "premise_reliability": {           "status": "pass|fail|conditional",           "issues": ["weak_evidence_for_A1"]         },         "reality_consistency": {           "status": "pass|fail|conditional",           "issues": ["conflicts_with_known_constraints"]         }       },       "overall_status": "pass|fail|conditional",       "fatal_issues": ["logical_invalidity"],       "notes": "검증 요약"     }   ] }`

---

## 2) 3계층 검증 구조 (핵심)

### Layer 1: Logical Validity (형식 논리)

**질문:**

> 전제가 참이라면, 결론(시나리오 해석)이 _논리적으로 따라오는가?_

#### 체크 항목

- 비약(non sequitur)
    
- 순환논증
    
- 전제 누락
    
- 내부 모순
    

#### 판정

- **fail** → 즉시 기각 후보
    
- conditional → 다음 레이어로 진행 가능 (감점)
    

---

### Layer 2: Premise Reliability (전제 신뢰성)

**질문:**

> 이 시나리오가 의존하는 전제들은 _근거 있는 전제_인가?

#### 체크 항목

- pivotal assumption의 근거 수준
    
- unresolved assumption 비중
    
- 출처 품질(Tier)
    
- 독립성
    

#### 판정

- pivotal 전제 fail → **치명적**
    
- 보조 전제 fail → conditional
    

---

### Layer 3: Reality Consistency (현실 일관성)

**질문:**

> 이 시나리오는 **이미 알려진 현실 제약**과 충돌하지 않는가?

#### 체크 항목

- 타임라인 충돌
    
- 제도/물리적 불가능성
    
- 과거 사례와의 구조적 모순
    
- 상식 위반
    

#### 판정

- 현실 충돌 → **fail**
    
- 불확실하지만 충돌은 아님 → conditional
    

---

## 3) 내부 처리 흐름 (서브스텝)

### 9-1. 시나리오 → 논리 구조화

각 시나리오를 다음 형태로 변환:

- 전제 집합
    
- 전제로부터 도출된 주장
    
- 결론(해석층)
    

> 이 구조화가 없으면 연역 검증 불가

---

### 9-2. Critique 반영

8단계 Cross-Critique 결과를 **검증 힌트**로 사용

- severity=high 이슈는 우선 검증
    
- pivotal assumption 연관 이슈는 가중치 ↑
    

---

### 9-3. 레이어별 독립 검증

각 레이어는 **독립적으로 pass/fail 판단**

- 한 레이어의 fail은 다른 레이어에서 상쇄 불가
    

---

### 9-4. Overall Status 결정

규칙:

- Logical fail → overall fail
    
- Reality fail → overall fail
    
- Premise conditional만 있는 경우 → overall conditional
    

---

## 4) 에이전트 책임과 금지사항

### Verifier Agent 책임

- 논리·전제·현실 검증 수행
    
- pass/fail/conditional 판정
    
- 이슈 타입 명시
    

### 금지사항

- 새로운 증거 생성 금지
    
- 시나리오 수정 금지
    
- “이게 더 낫다” 비교 금지
    

---

## 5) 품질 게이트 (이 단계의 합격 조건)

다음 중 하나 이상이 충족되어야 10단계로 진행:

1. pass 시나리오 ≥ 1
    
2. pass가 없더라도 conditional 시나리오 ≥ 1
    
3. 모든 시나리오가 fail인 경우 → **최소 위반 시나리오**를 conditional로 승격(단, 강한 경고 표시)
    

> **항상 판결 단계로 간다**는 원칙 유지

---

## 6) 운영 지표

- Pass rate / Conditional rate
    
- Fatal issue frequency
    
- Layer별 fail 분포
    
- Pivotal assumption fail rate
    

---

## 7) 이 단계의 결정적 의미

- **“그럴듯하지만 논리적으로 말이 안 되는 설명” 완전 제거**
    
- 판결이 “감”이 아니라 **연역적 자격**에 기반
    
- 다음 단계(Judge)가 “고민”이 아니라 **선택**만 하게 됨