**목적:**  
4단계에서 만든 **Evidence Map / Timeline이라는 ‘해석 가능 범위’ 안에서만**  
→ **후보 시나리오(3~5개)를 생성**한다.

> 핵심 전환점
> 
> - 여기서부터 “이야기”를 만들 수 있음
>     
> - 하지만 **근거 없는 문장은 구조적으로 불가능**
>     
> - 이 단계의 결과는 **판결 대상 후보**이지, 결론이 아님
>     

---

## 1) 입력과 출력

### 입력 (Evidence Map Pack)

4단계 출력 전체

`{   "timeline": [...],   "evidence_graph": {...},   "coverage_analysis": {...},   "risk_flags": [...] }`

---

### 출력 (Candidate Scenarios Pack)

이 출력은 **6단계(Pivotal Assumption Finder + Gap Detector)**의 입력이 된다.

`{   "candidate_scenarios": [     {       "scenario_id": "S1",       "title": "시나리오 한 줄 요약",       "observed_layer": [         {           "statement": "관측된 사실/사건",           "linked_event_ids": ["EV1"],           "evidence_ids": ["E1", "E2"]         }       ],       "interpretation_layer": [         {           "statement": "관측을 연결한 해석(최소)",           "based_on": ["observed_layer:1"],           "risk_flags": ["correlation_only"]         }       ],       "assumptions_layer": [         {           "assumption_id": "A1",           "statement": "근거 없는 연결 가정",           "assumption_cost": "low|medium|high",           "why_needed": "관측만으로는 설명 불가"         }       ],       "timeline_alignment": {         "consistent": true,         "issues": []       },       "known_weaknesses": [         "post_hoc_risk",         "data_gap_2025Q4"       ],       "testable_predictions": [         "추가로 확인되면 강화/약화될 관측 포인트"       ]     }   ],   "generation_notes": {     "scenario_count": 4,     "generation_constraints_applied": [       "no_causation_without_mechanism",       "timeline_consistency_required",       "assumption_penalty_active"     ]   } }`

---

## 2) 핵심 설계: ‘3층 구조’ 강제

모든 시나리오는 **반드시 3층으로 분리**된다.

### ① Observed Layer (관측층)

- Evidence Card / Event에 **직접 연결된 사실만**
    
- 평가·의미·원인 표현 금지
    

예:

- “2026-01-15에 정책 A가 발표됨”
    
- “지표 X가 12% 상승함”
    

---

### ② Interpretation Layer (해석층)

- 관측을 **최소한으로 연결**
    
- 인과 단정 금지
    
- `risk_flags` 필수
    

예:

- “정책 발표 이후 지표 상승이 관측됨 (상관관계)”
    

---

### ③ Assumptions Layer (가정층)

- 관측과 해석으로는 설명되지 않는 부분
    
- 반드시 **왜 필요한지 명시**
    
- 자동으로 비용(cost) 부과
    

> 이 층이 명시되어 있다는 것 자체가  
> “틀린 시나리오를 덜 만드는 장치”

---

## 3) 시나리오 생성 규칙 (강제)

### 규칙 1) 근거 없는 문장은 생성 불가

- evidence_ids 없는 문장은 **무조건 가정층으로 이동**
    
- 가정층에 쌓일수록 자동 감점
    

---

### 규칙 2) 타임라인 위반 즉시 폐기

- 사건 선후가 맞지 않으면 **시나리오 생성 중단**
    
- `timeline_alignment.consistent == false`면 제외
    

---

### 규칙 3) 인과 단정 금지

- causes / leads to / results in → 사용 불가
    
- 허용: “이후 발생”, “동시에 관측”, “연관 가능성”
    

---

### 규칙 4) 시나리오 다양성 확보

서로 다른 관점 최소 3개:

- 구조적/제도적
    
- 행위자/의도 기반
    
- 외생 변수/환경 요인
    

단, **근거 없는 상상은 금지**

---

## 4) 내부 처리 흐름 (서브스텝)

### 5-1. Evidence Map에서 해석 가능한 패턴 추출

- 반복 사건
    
- 급격한 변화
    
- 정책–지표 동시성
    
- 대응/반응 이벤트
    

### 5-2. 각 패턴을 설명할 수 있는 “최소 설명” 구성

- 최대 설명 ❌
    
- **최소 설명 ✅**
    

### 5-3. 설명 불가 구간 → 가정으로 명시

- 자동으로 assumptions_layer로 이동
    
- 비용 부과
    

---

## 5) 에이전트 책임과 금지사항

### Scenario Generator Agent 책임

- Evidence Map 범위 내에서만 시나리오 생성
    
- 3층 구조 준수
    
- 약점/리스크 공개
    

### 금지사항

- 새로운 사실 발명
    
- Evidence Card 없는 주장 생성
    
- “가장 그럴듯하다” 평가
    

---

## 6) 품질 게이트 (이 단계의 합격 조건)

다음 조건을 만족해야 6단계로 진행:

1. 시나리오 ≥ 3개
    
2. 모든 시나리오가 3층 구조를 가짐
    
3. 모든 해석이 timeline과 충돌하지 않음
    
4. 가정층이 명시되어 있음
    

---

## 7) 운영 지표

- Assumption load: 시나리오별 가정 수
    
- Evidence density: 관측 문장당 evidence 수
    
- Timeline violation rate: 생성 중 폐기 비율
    
- Scenario diversity score: 관점 중복도
    

---

## 이 단계의 결정적 의미

- **“이야기를 만들 수는 있지만, 상상은 못 한다”**
    
- 시나리오가 이미 **절반 이상 검증된 상태**로 출현
    
- 이후 단계는 “어느 게 덜 틀렸는지”를 고르는 문제로 단순화