**목적:**  
5단계에서 생성된 후보 시나리오들에 대해  
→ **판결을 갈라놓는 ‘결정적 전제(pivotal assumption)’를 식별**하고  
→ **추가 수집이 필요한 정보 공백을 정확히 지정**한다.

> 핵심 포인트
> 
> - “가정이 있다”는 것 자체는 문제 아님
>     
> - **어떤 가정이 결과를 바꾸는가**가 중요
>     
> - 이 단계는 **검색을 통제**하는 핵심 엔진
>     

---

## 1) 입력과 출력

### 입력 (Candidate Scenarios Pack)

5단계 출력 전체

`{   "candidate_scenarios": [...],   "generation_notes": {...} }`

---

### 출력 (Pivotal Analysis Pack)

이 출력은 **7단계(Targeted Retrieval Loop)** 의 입력이 된다.

`{   "pivotal_assumptions": [     {       "assumption_id": "A1",       "scenario_ids": ["S1", "S3"],       "statement": "이 가정이 참/거짓일 때 시나리오 우열이 바뀜",       "evidence_type_needed": "fact|stats|mechanism",       "current_support": "missing|weak|moderate|strong",       "discriminative_power": "high|medium|low",       "why_pivotal": "이 가정이 확인되면 S1이 S3보다 우세해짐"     }   ],   "scenario_dependency": [     {       "scenario_id": "S1",       "depends_on": ["A1", "A2"],       "robustness": "fragile|moderate|robust"     }   ],   "information_gaps": [     {       "gap_id": "G1",       "related_assumption_id": "A1",       "description": "전제 확인에 필요한 데이터 부재",       "evidence_type": "mechanism",       "priority": 1     }   ],   "retrieval_recommendation": {     "need_additional_retrieval": true,     "target_assumptions": ["A1"],     "expected_impact": "high"   } }`

---

## 2) 핵심 개념 정리

### Pivotal Assumption이란?

> **참/거짓 여부에 따라 최종 판결(단일 시나리오)이 바뀌는 가정**

- 모든 가정 ≠ pivotal
    
- “있으면 좋다” ❌
    
- “확인되면 판결이 갈린다” ⭕
    

---

## 3) 내부 처리 흐름 (서브스텝)

### 6-1. 시나리오별 가정 수집

각 시나리오의 `assumptions_layer`를 모두 수집한다.

---

### 6-2. 가정의 “판결 영향도” 평가

각 가정에 대해 질문한다:

- 이 가정이 참이면?
    
- 거짓이면?
    
- **시나리오 간 상대 순위가 바뀌는가?**
    

바뀌면 → **pivotal**

---

### 6-3. Discriminative Power 계산

가정의 판별력을 평가한다.

- **High**: 이 가정 하나로 S1 vs S2 판결 가능
    
- **Medium**: 여러 가정 중 하나
    
- **Low**: 보조적 정보
    

---

### 6-4. 현재 근거 수준 평가

Evidence Pack을 다시 조회하여:

- 해당 가정을 직접 지지/반박하는 evidence가 있는가?
    
- 있다면 강도는?
    

결과:

- missing / weak / moderate / strong
    

---

### 6-5. 정보 공백(Gap) 정의

pivotal assumption 중

- `current_support == missing | weak` 인 것만
    
- **추가 검색 대상**으로 정의
    

---

### 6-6. Retrieval Recommendation 생성

- 추가 수집 필요 여부
    
- 우선순위(1~N)
    
- 기대 효과(판결 분기 해결 가능성)
    

> 이 출력이 다음 단계에서 **검색 루프를 ‘정밀하게’ 작동**시킨다.

---

## 4) 에이전트 책임과 금지사항

### Pivotal Assumption Finder 책임

- 가정의 영향도 평가
    
- pivotal 여부 판정
    
- 검색 필요성 명시
    

### 금지사항

- 새로운 시나리오 생성 금지
    
- 가정이 참/거짓이라는 판단 금지
    
- “이 시나리오가 맞다” 결론 금지
    

---

## 5) 품질 게이트 (이 단계의 합격 조건)

다음 조건을 만족해야 7단계로 진행:

1. 최소 1개의 pivotal assumption 식별
    
2. 각 pivotal assumption에 evidence_type_needed 지정
    
3. 추가 검색 필요 여부가 명시됨
    
4. 시나리오별 robustness 평가가 존재
    

---

## 6) 운영 지표

- Pivotal ratio: 전체 가정 중 pivotal 비율
    
- Gap resolution rate: 이후 단계에서 해결되는 비율
    
- Over-retrieval rate: 불필요한 검색 추천 비율
    
- Decision sensitivity: pivotal 1개로 판결이 갈리는 빈도
    

---

## 이 단계의 결정적 의미

- **검색이 목적 없이 늘어나는 것을 차단**
    
- “조금 더 찾으면 될 것 같다”를 **정확한 대상**으로 환원
    
- 이후 판결이 **운이 아니라 정보에 의해 갈리도록 보장**