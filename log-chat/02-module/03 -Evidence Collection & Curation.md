**목적:** 2단계(Data Requirements Spec)에서 정의한 요구사항을 기반으로,  
**신뢰 가능한 데이터·사건·문서를 수집하고 정제하여 ‘시나리오 생성이 가능한 근거 풀’을 만든다.**

> 핵심 포인트
> 
> - 이 단계가 끝나기 전까지 **시나리오 생성은 금지**
>     
> - 모든 근거는 **Evidence Card**로만 존재
>     
> - “많이 모으기”보다 **품질·구조·연결성**이 중요
>     

---

## 1) 입력과 출력

### 입력 (Data Requirements Spec)

2단계 출력 전체

```json
{
  "decision_target": "...",
  "requirements": [...],
  "retrieval_policy": {...},
  "tier_policy": {...},
  "stop_conditions": [...]
}
```

---

### 출력 (Curated Evidence Pack)

이 출력은 **4단계(Evidence Map / Timeline Builder)** 의 입력이 된다.

```json
{
  "evidence_cards": [
    {
      "evidence_id": "E1",
      "type": "event|stat|document|statement|research",
      "summary": "한 문장 요약",
      "source": {
        "name": "출처명",
        "url": "링크",
        "tier": "T1|T2|T3",
        "publisher": "기관/언론/개인",
        "independence_group": "동일 출처 군"
      },
      "date": {
        "published": "YYYY-MM-DD",
        "event_date": "YYYY-MM-DD or range"
      },
      "content_excerpt": "핵심 원문 인용 (짧게)",
      "linked_claim_ids": ["C1"],
      "linked_requirement_ids": ["R1"],
      "stance": "supports|refutes|neutral|unclear",
      "strength": "weak|moderate|strong",
      "quality_flags": [
        "primary_source",
        "official_record",
        "translation_risk",
        "secondary_reporting"
      ],
      "notes": "주의사항/해석 제한"
    }
  ],
  "coverage_report": {
    "requirements_covered": ["R1", "R2"],
    "requirements_missing": ["R3"],
    "tier_summary": {
      "T1": 4,
      "T2": 3,
      "T3": 1
    }
  },
  "retrieval_log": [
    {
      "query": "검색 질의",
      "reason": "R1 충족",
      "result_count": 5,
      "stop_reason": "T1 확보"
    }
  ]
}
```

---

## 2) 내부 처리 흐름 (서브스텝)

### 3-1. Breadth-first Seed Collection (1차 수집)

**목표:**

- 주제의 **사건 골격 + 공식 사실 + 기본 수치** 확보
    
- “이 주제에서 뭘 모르면 안 되는지”의 최소 커버리지 확보
    

#### 수행 규칙

- 각 requirement에 대해:
    
    - T1 출처 우선
        
    - 실패 시 T2로 보완
        
- 동일 출처군(independence_group)은 **1개로 계산**
    

#### 결과물

- 사건 리스트(날짜 포함)
    
- 공식 발표/공시/문서
    
- 핵심 통계 지표
    

> 이 단계가 끝나면 “아무것도 없는 상태에서 상상으로 만드는 것”은 불가능해짐

---

### 3-2. Evidence Card 생성 (강제 포맷)

모든 수집 결과는 **반드시 Evidence Card**로 변환한다.

#### 금지사항

- 문단 통째 저장 금지
    
- 출처 없는 요약 금지
    
- “~라고 한다” 식 간접 인용 금지
    

Evidence Card는:

- **무엇이 있었는지**
    
- **언제 있었는지**
    
- **누가/어디서 말했는지**
    
- **무엇을 지지/반박하는지**
    

만 남긴다.

---

### 3-3. 품질 평가 & Tier 적용

수집 직후 **자동 품질 평가**를 수행한다.

#### 평가 항목

- 출처 Tier (T1/T2/T3)
    
- 원문 여부
    
- 날짜 명시 여부
    
- 독립성(같은 뉴스의 재전재인지)
    

#### 자동 플래그 예

- `secondary_reporting`
    
- `translation_risk`
    
- `date_ambiguity`
    
- `source_concentration`
    

이 플래그는 **후속 단계에서 감점/경고**로 사용된다.

---

### 3-4. Coverage & Gap Check

Data Requirements Spec과 비교하여:

- 어떤 requirement가 충족되었는지
    
- 어떤 requirement가 미충족인지
    
- 미충족 사유는:
    
    - 검색 실패
        
    - 출처 부재
        
    - 시간 불명
        
    - 충돌 증거 존재
        

를 명확히 기록한다.

---

### 3-5. Stop Conditions 평가

2단계에서 정의한 stop_conditions를 평가한다.

- 충족 → 4단계로 진행
    
- 미충족 → retrieval_policy에 따라:
    
    - 추가 seed 검색
        
    - or “미충족 상태”를 명시하고 다음 단계로 진행  
        (※ **다음 단계에서 가정 비용이 증가**)
        

---

## 3) 에이전트 책임과 금지사항

### Evidence Collector & Curator Agent 책임

- 검색 수행
    
- Evidence Card 생성
    
- 품질 플래그 부여
    
- coverage_report 작성
    

### 금지사항

- 시나리오/원인/해석 생성 금지
    
- 여러 evidence를 묶어서 결론 내리기 금지
    
- “이건 중요한 것 같다” 같은 주관 평가 금지
    

---

## 4) 품질 게이트 (이 단계의 합격 조건)

다음 중 **최소 조건**을 만족해야 다음 단계로 이동:

1. 핵심 requirement의 **T1 Evidence Card ≥ 1**
    
2. 핵심 requirement의 **독립 출처 ≥ 2**
    
3. 사건형 데이터의 경우 날짜가 명시됨
    
4. coverage_report가 생성됨
    

※ 불충족 시에도 **강제 종료는 하지 않음**  
→ 단, 이후 단계에서 **가정 페널티**로 반영

---

## 5) 운영 지표 (실제 중요한 것들)

- Evidence density: claim당 evidence 수
    
- Tier ratio: T1 비율
    
- Independence ratio: 출처 다양성
    
- Time coverage: 타임라인 공백 비율
    
- Noise rate: 최종 사용되지 않는 evidence 비율
    

---

## 6) 최소 프롬프트 템플릿 (Evidence Collector Agent)

- Data Requirements Spec을 읽고 검색을 수행하라
    
- 모든 결과를 Evidence Card로 변환하라
    
- 출처/날짜/원문 인용을 반드시 포함하라
    
- 판단·추론·결론을 절대 생성하지 말라
    
- 미충족 requirement와 이유를 명시하라
    

---

## 이 단계가 시스템에 주는 결정적 효과

- **근거 없는 시나리오 생성 불가능**
    
- 타임라인 오류/가짜 인과 급감
    
- “그럴듯한 이야기”가 아니라 **자료 위의 이야기**만 남음
    

---

