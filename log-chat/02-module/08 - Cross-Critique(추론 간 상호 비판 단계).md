**목적:**  
7단계까지 확보된 **근거 + 해결/미해결 전제 상태**를 바탕으로  
→ **후보 시나리오들이 서로의 논리·근거·가정을 직접 공격**하게 하여  
→ 연역 검증 전에 이미 **취약한 시나리오를 드러낸다**.

> 핵심 포인트
> 
> - 이 단계는 “판결”이 아니다
>     
> - **서로의 약점을 노출**시키는 단계
>     
> - 약점이 드러난 시나리오는 이후 단계에서 자동 감점·기각 후보
>     

---

## 1) 입력과 출력

### 입력 (Candidate Scenarios + Evidence Updates)

5·7단계의 통합 결과

`{   "candidate_scenarios": [...],   "updated_evidence_cards": [...],   "resolution_status": [...],   "unresolved_assumptions": [...] }`

---

### 출력 (Cross-Critique Pack)

이 출력은 **9단계(3-Layer Deductive Verification)**의 입력이 된다.

`{   "critiques": [     {       "critic_scenario_id": "S2",       "target_scenario_id": "S1",       "critique_type": "evidence_insufficiency|false_analogy|hidden_assumption|base_rate_neglect|timeline_violation|mechanism_gap",       "statement": "비판 요지 (한 문장)",       "supporting_points": [         "구체적 근거 1",         "구체적 근거 2"       ],       "severity": "low|medium|high",       "linked_assumption_ids": ["A1"],       "linked_evidence_ids": ["E3"]     }   ],   "scenario_vulnerability": [     {       "scenario_id": "S1",       "major_issues": ["hidden_assumption", "mechanism_gap"],       "vulnerability_score": 0.62     }   ],   "consensus_signals": [     {       "scenario_id": "S3",       "supporting_scenarios": ["S1", "S4"],       "reason": "공통 근거/전제 공유"     }   ] }`

---

## 2) 비판 유형 표준화 (중요)

Cross-Critique는 **감정/표현이 아니라 타입**으로 한다.

### 허용 비판 유형

- `evidence_insufficiency`: 근거 부족
    
- `false_analogy`: 유사 사례 오용
    
- `hidden_assumption`: 숨은 가정
    
- `base_rate_neglect`: 기초율 무시
    
- `timeline_violation`: 시간 선후 위반
    
- `mechanism_gap`: 인과 메커니즘 부재
    
- `overfitting`: 특정 사건에 과도 의존
    
- `confirmation_bias`: 지지 증거만 선택
    

> 이 타입은 이후 **Failure Ontology**와 직접 연결됨

---

## 3) 내부 처리 흐름 (서브스텝)

### 8-1. 모든 시나리오 쌍(pairwise) 비교

- 자기 자신 제외
    
- S_i → S_j 형태로 비판 생성
    

---

### 8-2. “치명적 약점” 우선 탐지

다음에 해당하면 **severity=high**:

- unresolved pivotal assumption에 전적으로 의존
    
- 타임라인 위반
    
- 인과 주장의 근거가 전무
    
- 근거가 단일 출처군에 집중
    

---

### 8-3. Consensus Signal 탐지

의외로 중요한 기능:

- 서로 다른 시나리오가
    
- **같은 근거/전제**를 독립적으로 사용하면
    
- “공통 기반 신호”로 기록
    

> 이후 판결 단계에서 **가중치 보정**에 사용

---

### 8-4. Vulnerability Score 계산

각 시나리오에 대해:

- 비판 개수
    
- severity 가중치
    
- pivotal assumption 연관 여부
    
- 해결/미해결 전제 상태
    

를 합산해 **취약성 점수(0~1)** 산출

---

## 4) 에이전트 책임과 금지사항

### Cross-Critique Agent 책임

- 구조화된 비판 생성
    
- severity 부여
    
- consensus 탐지
    

### 금지사항

- “이 시나리오가 맞다” 판단 금지
    
- 새로운 증거 생성 금지
    
- 비판을 이유로 시나리오 삭제 금지
    

---

## 5) 품질 게이트 (이 단계의 합격 조건)

다음 조건을 만족하면 9단계로 진행:

1. 모든 시나리오에 대해 최소 1개 이상의 critique 존재
    
2. severity가 high인 이슈가 명시됨
    
3. vulnerability_score가 계산됨
    

---

## 6) 운영 지표

- Avg critiques per scenario
    
- High severity rate
    
- Consensus frequency
    
- Vulnerability dispersion (시나리오 간 편차)
    

---

## 7) 이 단계의 결정적 의미

- **검증 전에 이미 “약한 놈”이 드러남**
    
- 연역 검증이 “모두를 다 검사”할 필요 감소
    
- 이후 단계의 판결이 **갑툭튀가 아니라 누적 결과**가 됨
    

---