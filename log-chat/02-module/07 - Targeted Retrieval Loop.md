**(결정적 전제 전용 추가 수집)**

**목적:**  
6단계에서 식별된 **pivotal assumption(결정적 전제)**에 대해서만  
→ **정밀·집중 검색을 수행하여 판결 분기를 해결하거나, 해결 불가함을 명시**한다.

> 핵심 원칙
> 
> - 검색은 **판결을 바꾸는 전제**에만 수행
>     
> - “있으면 좋다” 정보는 수집하지 않음
>     
> - 해결되지 않아도 **판결은 반드시 내려야 함**
>     

---

## 1) 입력과 출력

### 입력 (Pivotal Analysis Pack)

6단계 출력 전체

`{   "pivotal_assumptions": [...],   "information_gaps": [...],   "retrieval_recommendation": {...} }`

---

### 출력 (Targeted Evidence Update Pack)

이 출력은 **8단계(Cross-Critique)**와 **9단계(3-Layer Verification)**에 직접 전달된다.

`{   "retrieved_evidence_cards": [     {       "evidence_id": "E_new_1",       "type": "fact|stat|mechanism",       "summary": "결정적 전제 관련 핵심 내용",       "source": {         "name": "출처명",         "url": "링크",         "tier": "T1|T2",         "publisher": "기관/학회/정부",         "independence_group": "그룹ID"       },       "date": {         "published": "YYYY-MM-DD",         "event_date": "YYYY-MM-DD or range"       },       "content_excerpt": "짧은 원문 인용",       "linked_assumption_ids": ["A1"],       "stance": "supports|refutes|neutral",       "strength": "weak|moderate|strong",       "quality_flags": [         "primary_source",         "peer_reviewed",         "official_record"       ],       "notes": "해석 시 주의사항"     }   ],   "resolution_status": [     {       "assumption_id": "A1",       "before_support": "missing",       "after_support": "moderate",       "resolved": true     }   ],   "retrieval_log": [     {       "assumption_id": "A1",       "query": "검색 질의",       "result_count": 3,       "stop_reason": "T1 evidence 확보"     }   ],   "unresolved_assumptions": [     {       "assumption_id": "A2",       "reason": "관련 1차 출처 부재",       "impact": "판결에 불확실성 잔존"     }   ] }`

---

## 2) 내부 처리 흐름 (서브스텝)

### 7-1. 대상 전제 선정

- `retrieval_recommendation.target_assumptions`만 처리
    
- priority 순으로 수행
    
- **비결정적 가정은 무시**
    

---

### 7-2. Evidence Type별 검색 전략

전제에 따라 검색 전략을 달리한다.

#### Fact

- 공식 발표
    
- 공시
    
- 원문 발언/문서
    

#### Stats

- 정부/공식 통계
    
- 장기 시계열
    
- 표본 정의 확인
    

#### Mechanism

- 논문
    
- 기술 문서
    
- 제도 설명서
    
- 전문가 합의 문서
    

---

### 7-3. 품질 필터 (강제)

- T3 출처는 기본적으로 제외
    
- T2는 T1 실패 시에만 허용
    
- 동일 출처군은 1개로 계산
    

---

### 7-4. Stop Condition 평가

다음 중 하나면 해당 전제에 대한 검색 종료:

- T1 evidence 1개 확보
    
- 독립 출처 2개 확보
    
- 상충 evidence가 발견되어 추가 검색이 무의미
    
- 검색 예산 초과
    

---

### 7-5. Resolution 판정

검색 후:

- support 수준 갱신
    
- resolved = true/false 설정
    
- unresolved는 **명시적으로 남김**
    

> 해결되지 않은 전제는 이후 단계에서  
> **신뢰도 감점 + 리스크 명시**로 반영

---

## 3) 에이전트 책임과 금지사항

### Targeted Retrieval Agent 책임

- 결정적 전제에 한정한 검색 수행
    
- Evidence Card 생성
    
- resolution_status 업데이트
    

### 금지사항

- 시나리오 수정 금지
    
- 가정이 참/거짓이라는 결론 금지
    
- 검색 결과를 해석해서 연결 금지
    

---

## 4) 품질 게이트 (이 단계의 합격 조건)

다음 조건 중 하나를 만족하면 다음 단계로 진행:

1. 모든 pivotal assumption이 resolved
    
2. 더 이상의 검색이 판결을 바꾸지 못한다고 판단됨
    
3. 검색 예산 초과 (단, unresolved 명시 필수)
    

---

## 5) 운영 지표

- Resolution rate: pivotal assumption 해결률
    
- Retrieval efficiency: 전제당 검색 수
    
- Evidence yield: 검색 대비 유효 evidence 비율
    
- Residual uncertainty: 미해결 전제 비율
    

---

## 6) 이 단계의 결정적 의미

- **“조금만 더 찾으면…” 루프를 강제 종료**
    
- 판결이 **정보 부족인지, 정보의 결과인지**를 명확히 구분
    
- 이후 단계의 논쟁은 “해석”이 아니라 **판결 품질**로 이동