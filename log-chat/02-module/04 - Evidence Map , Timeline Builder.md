**목적:** 3단계에서 수집·정제된 Evidence Card들을  
**시간·사건·관계 구조로 재조합**하여  
→ **잘못된 인과, 거짓 유추, 중요한 누락을 사전에 차단**한다.

> 핵심 포인트
> 
> - 아직도 **시나리오 생성 금지**
>     
> - 이 단계의 출력은 “이야기”가 아니라 **지도(Map)**
>     
> - 이후 단계에서 **해석 가능한 범위가 여기서 결정됨**
>     

---

## 1) 입력과 출력

### 입력 (Curated Evidence Pack)

3단계 출력 전체

`{   "evidence_cards": [...],   "coverage_report": {...},   "retrieval_log": [...] }`

---

### 출력 (Evidence Map Pack)

이 출력은 **5단계(Evidence-Constrained Scenario Generator)**의 입력이 된다.

`{   "timeline": [     {       "time": "YYYY-MM-DD",       "event_id": "EV1",       "description": "사건/발언/공시 요약",       "linked_evidence_ids": ["E1", "E3"],       "entities": ["엔티티A", "엔티티B"],       "type": "event|statement|policy|data_release",       "confidence": "high|medium|low",       "notes": "해석 금지, 사실적 서술만"     }   ],   "evidence_graph": {     "nodes": [       {         "node_id": "N1",         "type": "event|entity|document|metric",         "label": "노드명"       }     ],     "edges": [       {         "from": "N1",         "to": "N2",         "relation": "temporal_precedes|references|responds_to|correlates_with",         "evidence_ids": ["E2"]       }     ]   },   "coverage_analysis": {     "claim_coverage": [       {         "claim_id": "C1",         "covered_by_events": ["EV1", "EV2"],         "coverage_quality": "strong|partial|weak|missing"       }     ],     "timeline_gaps": [       {         "gap_period": "YYYY-MM-DD ~ YYYY-MM-DD",         "risk": "potential_missing_event"       }     ]   },   "risk_flags": [     "post_hoc_risk",     "correlation_only",     "event_overlap_confusion"   ] }`

---

## 2) 내부 처리 흐름 (서브스텝)

### 4-1. 시간 정규화 (Temporal Normalization)

Evidence Card의 날짜 정보를 정규화한다.

- `event_date` > `published_date` > `unknown`
    
- 날짜 범위는 range로 유지
    
- 불명확하면 `low confidence`로 표시
    

> 이 단계에서 **시간 없는 근거는 자동 감점** 대상이 됨

---

### 4-2. 사건(Event) 추출

Evidence Card들을 **사건 단위**로 묶는다.

사건이란:

- 특정 날짜(또는 기간)에
    
- 특정 엔티티가
    
- 특정 행위를 한 것
    

예:

- 정책 발표
    
- 수치 발표
    
- 발언
    
- 사건 발생
    

※ 하나의 사건에 여러 Evidence Card가 연결될 수 있음

---

### 4-3. Timeline 구성

사건들을 날짜 기준으로 정렬한다.

규칙:

- 해석 금지
    
- “A 때문에 B” 표현 금지
    
- 오직 “A 이후 B 발생”만 허용
    

이 타임라인은 이후 단계에서 **인과 주장에 대한 제약 조건**으로 작동한다.

---

### 4-4. Evidence Graph 구성 (관계 지도)

시간 외의 관계를 명시적으로 분리한다.

허용 관계:

- `temporal_precedes` (시간상 선행)
    
- `references` (문서/발언 참조)
    
- `responds_to` (대응)
    
- `correlates_with` (상관)
    

금지 관계:

- causes
    
- leads_to
    
- results_in
    

> 인과는 **아직 만들지 않는다**  
> 인과를 만들 수 있는 **재료만** 준비한다.

---

### 4-5. Coverage & Gap 분석

Data Requirements Spec의 claim들과 비교하여:

- 어떤 claim이 충분히 사건으로 커버되는지
    
- 어떤 claim에 **사건 공백**이 있는지
    
- 어떤 기간에 **중요 이벤트 누락 가능성**이 있는지
    

이 결과는 다음 단계에서:

- 추가 검색 루프를 트리거하거나
    
- 시나리오 생성 시 **가정 비용**으로 반영된다.
    

---

### 4-6. 인과 오판 리스크 플래깅

다음과 같은 패턴을 자동 감지하여 플래그로 남긴다.

- `post_hoc_risk`: A 다음 B만 있고 연결 증거 없음
    
- `correlation_only`: 통계 상관만 존재
    
- `event_overlap_confusion`: 여러 사건이 겹쳐 원인 분리 불가
    

> 이 플래그는 5~9단계 전반에서 **강한 경고 신호**로 작동

---

## 3) 에이전트 책임과 금지사항

### Evidence Map / Timeline Builder Agent 책임

- 사건 추출
    
- 시간 정렬
    
- 관계 그래프 구성
    
- 누락/공백/리스크 표시
    

### 금지사항

- 인과 해석 금지
    
- 시나리오 생성 금지
    
- “중요하다/영향을 미쳤다” 같은 평가 금지
    

---

## 4) 품질 게이트 (이 단계의 합격 조건)

다음 조건을 충족해야 5단계로 진행:

1. 핵심 사건들이 날짜 순으로 정렬됨
    
2. 주요 claim에 대해 사건 매핑이 존재
    
3. timeline_gaps 또는 risk_flags가 명시됨
    
4. 관계 그래프에 **인과 관계가 없음**
    

---

## 5) 운영 지표

- Timeline completeness: 핵심 기간 커버율
    
- Event density: 기간 대비 사건 수
    
- Time ambiguity rate: 날짜 불명 사건 비율
    
- Risk flag frequency: 인과 오판 위험 비율
    

---

## 6) 최소 프롬프트 템플릿 (Evidence Map Builder)

- Evidence Card들을 사건 단위로 묶어라
    
- 날짜 기준으로 정렬하라
    
- 인과 해석을 절대 하지 말라
    
- 관계는 시간/참조/대응/상관까지만 허용하라
    
- 사건 공백과 인과 오판 위험을 플래그로 표시하라
    

---

## 이 단계의 결정적 의미

- **“사실은 맞는데 해석이 틀린 문제”를 사전에 차단**
    
- 가짜 인과, 과잉 유추, 서사 과장을 구조적으로 억제
    
- 다음 단계에서 **허용되는 해석 범위**를 강하게 제한