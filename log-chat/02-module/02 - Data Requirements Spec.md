**목적:** 1단계(Decomposer)의 구조화 결과를 받아, **“단일 시나리오 수렴에 필요한 데이터/사건/문서/지표를 무엇부터 어떤 품질로 수집할지”**를 _명세서 형태로 확정_한다.  
**이 단계에서 절대 하면 안 되는 것:** 시나리오 생성, 결론 도출, 사실 단정.

---

## 1) 입력과 출력

### 입력 (Decomposed Pack)

1단계 출력 그대로를 입력으로 받는다.

```json
{
  "topic": {...},
  "claims": [...],
  "question_types": [...],
  "constraints": {...},
  "risk_flags": [...]
}
```

### 출력 (Data Requirements Spec)

이 출력은 **3단계(Evidence Collection & Curation)**의 작업지시서가 된다.

```json
{
  "topic": {
    "title": "한 줄 주제명",
    "time_window": "unknown or YYYY-MM~YYYY-MM",
    "locale": "unknown or KR/US/..."
  },
  "objective": {
    "deliverable": "단일 시나리오 수렴 보고서",
    "decision_target": "무엇을 판결해야 하는가(핵심 질문 1개)"
  },
  "requirements": [
    {
      "req_id": "R1",
      "linked_claim_ids": ["C1", "C3"],
      "priority": 1,
      "evidence_type": "fact|stats|mechanism",
      "data_items": [
        {
          "item_id": "R1-I1",
          "description": "필요한 데이터/사건/문서 설명",
          "target_entity": "엔티티",
          "time_range": "필요 기간",
          "preferred_sources": [
            {"tier": "T1", "source_class": "공식문서/공시/정부통계/원문발언"},
            {"tier": "T2", "source_class": "주요 언론/전문기관 리포트"},
            {"tier": "T3", "source_class": "블로그/커뮤니티(기본 비권장)"}
          ],
          "min_quality_bar": {
            "must_have": ["T1 1개 이상", "독립 출처 2개 이상"],
            "disallow": ["출처 불명", "2차 재전재만", "날짜 불명"]
          },
          "acceptance_criteria": [
            "사건/수치/문서가 날짜와 함께 확인될 것",
            "원문 또는 1차 출처 링크가 존재할 것"
          ],
          "failure_mode": [
            "동명이인/오역/날짜 혼동 가능"
          ]
        }
      ]
    }
  ],
  "retrieval_policy": {
    "breadth_first_seed": {
      "enabled": true,
      "budget": {"queries": 8, "sources": 12},
      "goal": "주요 사건/사실/문서의 기본 커버리지 확보"
    },
    "pivotal_targeted_followup": {
      "enabled": true,
      "budget": {"queries": 6, "sources": 8},
      "goal": "결정적 전제(pivotal) 분기 해결"
    }
  },
  "tier_policy": {
    "T1": "공식/원문/공시/정부통계/법령/논문 1차",
    "T2": "주요 언론/전문기관 분석/학회 리뷰 등",
    "T3": "커뮤니티/블로그/재전재(참고용)"
  },
  "stop_conditions": [
    "핵심 claim별 T1 근거 1개 이상 확보",
    "핵심 claim별 독립 출처 2개 이상 확보",
    "시간축(timeline)의 주요 이벤트 공백이 해소됨"
  ],
  "open_questions": [
    "현재 입력에서 시간 범위가 불명확함 → seed 수집으로 추정 필요"
  ],
  "risks": [
    "high_polarization → 출처 품질 기준 강화",
    "insufficient_time_ref → 타임라인 우선 수집"
  ]
}
```

---

## 2) 내부 처리 흐름 (서브스텝)

### 2-1. “판결해야 할 단일 질문”을 1개로 고정

여기서부터 “단일 시나리오 수렴”을 위해 중심축이 필요합니다.

- 여러 claim이 있어도 **결국 판결해야 할 핵심 질문을 1개로 묶음**
    
- 예: “A 정책이 B 결과를 초래했는가?” / “이 사건의 가장 근거 기반 설명은 무엇인가?”
    

출력: `decision_target`

> 주의: “답”을 내지 않는다. “무엇을 판결할지”만 정한다.

---

### 2-2. claim → evidence_type 매핑 확정 (1단계는 예비, 2단계는 확정)

- fact → **fact** (원문/공식/공시 중심)
    
- trend → **stats + fact** (통계 출처 + 수치)
    
- causal → **mechanism + fact** (선후관계 + 작동 원리)
    
- policy → **fact(원문 문서) + mechanism**
    
- forecast → **stats(base rate) + mechanism**
    

---

### 2-3. 데이터 요구사항을 “수집 가능한 아이템”으로 쪼갬

각 claim을 만족시키려면 실제로 어떤 “수집 단위”가 필요한지 정의합니다.

예시(구조):

- 사건: 언제/어디서/누가/무슨 일이
    
- 수치: 지표명/단위/기간/출처
    
- 문서: 원문 링크/발행기관/발행일/핵심 조항
    
- 발언: 원문 발언/날짜/장소/맥락
    

이때 **수집 단위(item)**는 항상 다음을 포함:

- 엔티티
    
- 시간 범위
    
- 출처 등급(Tier)
    
- 합격 조건(acceptance_criteria)
    

---

### 2-4. Tier 정책(출처 품질) 고정

이 단계에서 출처 품질 규칙을 시스템 규칙으로 못 박아야 “틀린 것 생성 억제”가 가능합니다.

기본(권장) 정책:

- 핵심 claim은 **T1 최소 1개 + 독립 출처 2개**를 원칙으로
    
- T3는 단독 증거로 사용 금지(참고/단서만)
    

---

### 2-5. 수집 전략을 2단계로 나눔 (필수)

검색이 산만해지는 걸 막기 위해 정책을 분리합니다.

1. **Breadth-first seed 수집**
    

- 넓게: 주제의 주요 사건, 날짜, 공식 발표, 기본 통계 커버리지
    
- 목적: “타임라인/사건 지도” 기본 골격 확보
    

2. **Pivotal targeted follow-up**
    

- 좁게: 후보 시나리오들이 갈리는 “결정적 전제”만 핀포인트 수집
    
- 목적: 수렴에 필요한 분기 해결
    

> 2단계에서는 “정책만” 정의하고, 실제 실행은 3단계에서 수행.

---

### 2-6. Stop conditions 정의

언제 수집을 멈추고 다음 단계로 넘어갈지 명시해야 비용이 통제됩니다.

Stop 조건(권장):

- 핵심 claim별 T1 1개 이상
    
- 핵심 claim별 독립 출처 2개 이상
    
- 타임라인에서 “핵심 이벤트 공백”이 해소
    

---

## 3) 에이전트 책임과 금지사항

### Data Requirements Spec Agent 책임

- 수집해야 할 “데이터 목록/품질/우선순위/중단 조건”을 명세
    
- retrieval policy(예산/전략) 고정
    
- 출처 등급(tier) 정책 고정
    

### 금지사항

- 어떤 시나리오가 맞는지 언급 금지
    
- 특정 주장에 대해 “사실이다/아니다” 단정 금지
    
- 수집 결과를 상상해서 채우기 금지
    

---

## 4) 품질 게이트 (이 단계의 합격 조건)

다음 조건이 충족되면 3단계로 진행:

1. 핵심 `decision_target` 1개가 존재
    
2. 모든 주요 claim이 최소 1개의 requirement에 연결됨
    
3. 각 requirement가 최소 1개의 data_item을 가짐
    
4. 각 data_item에 `preferred_sources`, `min_quality_bar`, `acceptance_criteria` 존재
    
5. stop_conditions가 정의됨
    

---

## 5) 운영 지표 (디버깅/평가용)

- Requirement coverage: claim이 요구사항으로 잘 연결됐는가
    
- Over/Under-spec: 너무 많은 데이터 요구/너무 적은 요구
    
- Quality compliance: tier 정책이 실제로 지켜지는가(3단계에서 측정)
    
- Retrieval efficiency: seed vs follow-up 비중 적절한가
    

---

## 6) 최소 프롬프트 템플릿 (Data Requirements Spec Agent)

(설계 문서용)

- 입력 claim들을 “수집 가능한 데이터 아이템”으로 변환하라
    
- 핵심 판결 질문(decision_target)을 1개로 고정하라
    
- 각 데이터 아이템에 시간범위/엔티티/출처등급/합격기준을 부여하라
    
- 수집 전략을 seed(넓게)와 pivotal follow-up(좁게)로 분리하라
    
- stop_conditions를 명시하라
    
- 결론/시나리오/사실 단정 금지
    

---
