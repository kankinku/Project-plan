**목적:** 사용자가 말한 주제/주장을 **“수집 가능한 데이터 요구사항”으로 변환할 수 있도록** 입력을 구조화한다.  
**이 단계에서 절대 하면 안 되는 것:** 결론/시나리오 생성, 원인 단정, 사실 단정.

---

## 1) 단계의 입력과 출력

### 입력 (Raw Input)

- 사용자 텍스트(뉴스/이슈/주장/질문)
    
- (옵션) 사용자 메타: 날짜 힌트, 관심 범위, 도메인 힌트
    

`{   "text": "사용자 입력 원문",   "metadata": {     "timestamp_hint": "optional",     "domain_hint": "optional",     "locale_hint": "optional",     "user_intent_hint": "optional"   } }`

### 출력 (Decomposed Pack)

이 출력은 **2단계(Data Requirements Spec)** 로 바로 연결된다.

`{   "topic": {     "title": "한 줄 주제명",     "scope": "범위/시계열/지역 등"   },   "user_intent": {     "goal": "설명|검증|예측|비교|리스크평가|기타",     "deliverable": "단일 시나리오 수렴 보고서"   },   "claims": [     {       "claim_id": "C1",       "statement": "검증 가능한 주장 문장",       "type": "fact|causal|trend|policy|forecast|comparison|value_judgement",       "time_ref": "명시된 날짜/기간 또는 unknown",       "entities": ["엔티티1", "엔티티2"],       "required_evidence_type": ["fact|stats|mechanism"],       "ambiguities": ["모호한 용어", "정의 불명확 요소"],       "testable_questions": [         "이 주장은 사실인가?",         "어떤 지표로 확인 가능한가?"       ]     }   ],   "question_types": [     "causal_explanation",     "pattern_validation",     "historical_comparison",     "probabilistic_forecast",     "scenario_evaluation",     "system_interaction"   ],   "constraints": {     "must_include": ["반드시 다룰 요소"],     "must_exclude": ["제외할 요소"],     "time_window": "예: 2025-12~2026-01 또는 unknown",     "confidence_target": "high|medium|low (요구 신뢰 수준)"   },   "risk_flags": [     "misinfo_risk",     "high_polarization",     "insufficient_time_ref",     "entity_disambiguation_needed"   ] }`

---

## 2) 내부 처리 흐름 (서브스텝)

### 1-1. 텍스트 정규화

- 불필요한 줄바꿈/인용부호/링크 분리
    
- 링크는 **별도 배열**로 추출(추후 수집 단계에서 사용)
    

출력(내부):

- `clean_text`
    
- `links[]`
    

### 1-2. Topic 추출

- 주제를 “명사구 1개”로 고정
    
- 범위(scope)는 **최대한 보수적으로**(모르면 unknown)
    

### 1-3. Claim 분해 (가장 중요)

원문에서 “검증 가능한 주장”을 최소 단위로 쪼갠다.

- **fact**: 사건 발생/발언/수치
    
- **causal**: A가 B를 유발
    
- **trend**: 증가/감소/변동
    
- **policy**: 제도/규정/정책 변화
    
- **forecast**: 미래 발생 주장
    
- **comparison**: A가 B보다
    
- **value_judgement**: 좋다/나쁘다(검증불가 → 별도 취급)
    

> value_judgement는 버리진 않되, **“검증 불가 레이어”로 분리**해서 이후 단계에서 혼동 방지.

### 1-4. Question Types 결정

Claim들의 타입을 보고 질문 유형을 자동 도출한다.

예:

- causal claim 존재 → `causal_explanation`
    
- trend claim 존재 → `pattern_validation`
    
- comparison/유사사례 언급 → `historical_comparison`
    
- forecast 언급 → `probabilistic_forecast`
    

### 1-5. Required Evidence Type 예비 태깅

각 claim에 대해 “무슨 증거가 필요할지” 1차로만 태깅한다.

- fact → fact
    
- trend → stats + fact(데이터 출처)
    
- causal → mechanism + fact(선후관계)
    
- policy → fact(원문 문서) + mechanism(작동 원리)
    
- forecast → base rate/stats + mechanism
    

### 1-6. 모호성/리스크 플래그

- 날짜가 없으면 `insufficient_time_ref`
    
- 엔티티가 동명이인이면 `entity_disambiguation_needed`
    
- 정치/선동성 강하면 `high_polarization` / `misinfo_risk`
    

---

## 3) 에이전트 책임과 금지사항

### Decomposer Agent 책임

- 오직 **구조화**: topic, claims, question_types, ambiguities
    
- **추론/결론/평가 금지**
    

### 금지 출력 예시

- “원인은 ~~다” (금지)
    
- “가장 가능성 높은 시나리오는…” (금지)
    
- “이건 사실일 것이다” (금지)
    

---

## 4) 품질 게이트 (이 단계의 합격 조건)

다음 조건이 충족되어야 2단계로 넘어간다.

1. claims가 **최소 1개 이상**
    
2. 각 claim에 `type`이 부여됨
    
3. 각 claim에 `testable_questions`가 최소 1개
    
4. 시간 참조가 없으면 `risk_flags`에 표시됨
    

불합격 시:

- claim을 더 쪼개거나
    
- “unknown” 필드로라도 채워서 구조 완성(단, 결론은 금지)
    

---

## 5) 운영 지표 (디버깅/평가용)

- Claim coverage: 원문 핵심 주장 누락률
    
- Claim granularity: claim 1개가 너무 크지 않은가(평균 토큰/문장 수)
    
- Ambiguity rate: unknown/ambiguities 비율
    
- Downstream utility: 2단계에서 “요구 데이터 명세”가 자연스럽게 나오는지
    

---

## 6) 최소 프롬프트 템플릿 (Decomposer Agent)

(코드가 아니라 설계 문서용 템플릿)

- 입력을 읽고 **주제 1개**
    
- 검증 가능한 **claim을 최소 단위로 분해**
    
- claim마다 type, 필요한 증거 타입, 테스트 질문을 붙여라
    
- 모호성/날짜/엔티티 불명확은 risk_flags로 표시
    
- **결론/해석/원인 단정 금지**