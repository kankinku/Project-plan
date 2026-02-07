## Scenario JSON Specification (NL → JSON 분류/수정)

- **문서 ID**: SCHEMA-320
    
- **제목**: Scenario JSON Specification
    
- **버전**: 1.0
    
- **목표**: 사용자의 자연어 “앞으로 일어날 일(가설/시나리오)” 입력을 **검증 가능하고 전략에 반영 가능한 구조(JSON)** 로 변환한다.
    
- **범위**: 시나리오 구조화(분류/정규화/수정 제안/품질 점수) + 전략 반영을 위한 최소 필드.
    
- **비범위**: 시나리오의 사실 여부 판정(예측 정확도 경쟁). _본 스키마는 “가설 관리/검증 가능성” 목적._
    

---

## 1) 설계 원칙

1. **검증 가능성(Testability)**: 시간이/대상이/방향이 명시되어 백테스트/스트레스 테스트로 검증 가능한 형태로 만든다.
    
2. **불확실성 내장**: 확률/신뢰도/대안 시나리오를 구조에 포함한다.
    
3. **정규화/수정 제안**: 애매한 표현을 “측정 가능한 문장”으로 바꾸는 `recommended_edits`를 포함한다.
    
4. **전략 반영 가능**: 가설→영향 자산/팩터→제약/가중/테스트로 매핑 가능해야 한다.
    
5. **추적성**: 원문(raw_text)과 변환 결과의 관계가 명확해야 한다.
    

---

## 2) 최상위 구조(Top-level)

```json
{
  "schema_version": "1.0",
  "scenario_id": "uuid-or-hash",
  "created_at": "2026-02-06T00:00:00Z",
  "language": "ko",
  "raw_text": "사용자 원문",
  "source": "user|system|import",
  "context": {},
  "hypotheses": [],
  "alternatives": [],
  "recommended_edits": [],
  "quality_score": {},
  "integration_plan": {},
  "trace": {}
}
```

### 2.1 필수 필드

- `schema_version`
    
- `language`
    
- `raw_text`
    
- `hypotheses` (길이 ≥ 1)
    
- `quality_score` (최소 clarity/testability/consistency)
    

---

## 3) context (옵션, 환경/제약 정보)

시나리오를 해석할 때의 배경(자산군/시장/제약)을 명시.

```json
"context": {
  "asset_class_focus": ["equity", "etf"],
  "regions_focus": ["US", "KR"],
  "time_now": "2026-02-06",
  "user_constraints": {
    "no_leverage": true,
    "max_positions": 20
  }
}
```

---

## 4) hypotheses (핵심: 가설 목록)

`hypotheses`는 복수 가설을 허용한다(시나리오가 여러 주장으로 구성되는 경우).

```json
"hypotheses": [
  {
    "id": "H1",
    "claim": "정규화된 핵심 주장 문장",
    "time_horizon": { "start": "2026-02-01", "end": "2026-06-30", "confidence": 0.6 },
    "scope": {
      "regions": ["US"],
      "asset_classes": ["equity", "rates", "fx"],
      "tickers": ["QQQ"],
      "sectors": ["Semiconductors"],
      "themes": ["AI"]
    },
    "drivers": [],
    "expected_market_effects": [],
    "assumptions": [],
    "uncertainties": [],
    "evidence": [],
    "tags": ["macro", "rates", "risk_on"]
  }
]
```

### 4.1 hypotheses 필수 규칙

- `id`, `claim`, `time_horizon`, `scope`는 필수
    
- `time_horizon.start/end`는 ISO date(YYYY-MM-DD) 권장
    
    - 원문에 기간이 없으면 `recommended_edits`에 “기간 명시”를 추가하고, 임시로 `start = time_now`, `end = time_now + default_window`로 채우되 `confidence`를 낮게 둔다.
        

---

## 5) time_horizon

```json
"time_horizon": {
  "start": "2026-02-01",
  "end": "2026-06-30",
  "confidence": 0.6,
  "granularity": "day|week|month"
}
```

### 5.1 규칙

- `confidence`는 “이 기간 설정이 타당한 정도”(사실 확률이 아님)
    
- `granularity`는 백테스트 테스트 설계에 사용
    

---

## 6) scope (대상 범위)

```json
"scope": {
  "regions": ["US", "KR"],
  "asset_classes": ["equity", "crypto", "rates", "fx", "commodities"],
  "tickers": ["SPY", "TLT"],
  "sectors": ["Technology"],
  "themes": ["AI", "Energy"],
  "style_factors": ["growth", "value"],
  "market_caps": ["large", "mid", "small"]
}
```

### 6.1 규칙

- 가능한 한 **구체적 레벨**(ticker/sector)을 우선
    
- 구체 대상이 없으면 regions/asset_classes 수준이라도 반드시 채움
    

---

## 7) drivers (원인/촉발 요인)

원인은 구조화하여 “왜 그런 일이 일어난다고 보는지”를 기록한다.

```json
"drivers": [
  {
    "type": "macro|policy|earnings|liquidity|geopolitical|sentiment|supply_chain|tech",
    "factor": "policy_rate|inflation|usd|oil|earnings_growth|...",
    "direction": "up|down|widen|tighten|risk_on|risk_off",
    "magnitude": "small|medium|large",
    "confidence": 0.55,
    "notes": "근거 요약"
  }
]
```

---

## 8) expected_market_effects (시장 영향)

가설이 시장에 미칠 영향을 “대상/방향/강도/채널”로 기록.

```json
"expected_market_effects": [
  {
    "target": "equity|rates|fx|crypto|commodities|volatility",
    "segment": "ticker:QQQ|sector:Semiconductors|factor:growth|region:US",
    "direction": "up|down|steepen|flatten|risk_on|risk_off",
    "strength": 0.7,
    "channel": "discount_rate|earnings|liquidity|risk_premium|supply_demand",
    "time_lag": "0w|2w|1m|3m"
  }
]
```

### 8.1 규칙

- `strength`는 **영향 강도(0~1)**, 사실 확률이 아님
    
- `segment`는 가능한 표준 문자열 포맷 사용:
    
    - `ticker:XXX`, `sector:YYY`, `factor:ZZZ`, `region:AAA`
        

---

## 9) assumptions / uncertainties (전제와 불확실성)

### 9.1 assumptions

```json
"assumptions": [
  { "text": "전제 1", "criticality": "high|medium|low" }
]
```

### 9.2 uncertainties

```json
"uncertainties": [
  { "text": "불확실성 1", "impact": "high|medium|low" }
]
```

---

## 10) evidence (근거/출처 요약)

외부 링크/뉴스를 저장할 수도 있지만, MVP에서는 “요약 텍스트” 중심으로 둔다.

```json
"evidence": [
  {
    "type": "user_belief|historical_pattern|news|data_signal",
    "summary": "근거 요약(짧게)",
    "confidence": 0.4
  }
]
```

---

## 11) alternatives (대안 시나리오)

“같은 전제에서 다른 결과” 혹은 “정반대 결과”를 함께 기록(스트레스 테스트에 사용).

```json
"alternatives": [
  {
    "id": "A1",
    "relation": "inverse|branch|risk_case|bull_case|bear_case",
    "claim": "대안 주장",
    "time_horizon": { "start": "2026-02-01", "end": "2026-06-30", "confidence": 0.4 },
    "expected_market_effects": [
      { "target": "equity", "segment": "ticker:QQQ", "direction": "down", "strength": 0.6, "channel": "risk_premium" }
    ]
  }
]
```

---

## 12) recommended_edits (분류/수정 제안)

원문이 모호할 때 “검증 가능한 문장”으로 바꾸도록 제안.

```json
"recommended_edits": [
  {
    "type": "clarify_time|clarify_scope|quantify_magnitude|split_claim|resolve_contradiction",
    "before": "곧 금리 인하가 올 것이다",
    "after": "2026-02~2026-04 사이 25bp 인하 가능성을 가정한다",
    "reason": "기간/규모를 명시하면 백테스트 및 스트레스 테스트 설계 가능"
  }
]
```

---

## 13) quality_score (품질 점수)

“시나리오가 맞는지”가 아니라 “테스트 가능한지/일관적인지” 점수화.

```json
"quality_score": {
  "clarity": 0.7,
  "testability": 0.6,
  "internal_consistency": 0.8,
  "actionability": 0.65,
  "overall": 0.69
}
```

### 13.1 산출 규칙(권장)

- clarity: 기간/대상/방향의 명확성
    
- testability: 백테스트/지표로 관측 가능한 주장인지
    
- internal_consistency: 가설 간 충돌/모순 여부
    
- actionability: 전략 제약/가중/테스트로 연결 가능한 정도
    

---

## 14) integration_plan (전략 반영 계획)

시나리오를 “soft 방식”으로만 반영하기 위한 표준 출력.

```json
"integration_plan": {
  "mode": "soft",
  "constraints": [
    {
      "scope": "portfolio",
      "type": "SECTOR_TILT",
      "params": { "segment": "sector:Semiconductors", "tilt": "overweight", "max_sector_weight": 0.35 },
      "confidence": 0.5
    }
  ],
  "priority_weights": [
    { "target": "selection", "rule": "rs_percentile", "weight_delta": 0.1 }
  ],
  "stress_tests": [
    { "name": "scenario_inverse", "based_on": "A1", "required": true },
    { "name": "cost_shock", "params": { "slippage_bps": 2.0 }, "required": true }
  ],
  "notes": "하드 룰 변경 금지. 가중/제약/테스트로만 반영."
}
```

### 14.1 규칙

- `mode`는 기본 `soft`
    
- 반드시 `stress_tests`에 “반대 시나리오” 또는 “불리 케이스”를 최소 1개 포함 권장
    

---

## 15) trace (추적/감사)

변환 과정을 기록(디버그/재현용). MVP에서는 최소화.

```json
"trace": {
  "parser_version": "scenario_parser_v0",
  "normalizer_version": "scenario_norm_v0",
  "edit_rules_version": "scenario_edits_v0",
  "notes": "optional"
}
```

---

# 16) Validator 요구사항(필수 체크)

1. `hypotheses.length >= 1`
    
2. 각 hypothesis는 `id/claim/time_horizon/scope` 필수
    
3. `time_horizon.start <= time_horizon.end`
    
4. `quality_score` 4개 필드 필수 + `overall`은 0~1
    
5. `expected_market_effects[].strength`는 0~1
    
6. `integration_plan.mode == soft` 기본 (hard는 경고/금지 정책 가능)
    

---

# 17) 예시 2개

## 17.1 예시 A — 금리 인하(자연어 → JSON)

```json
{
  "schema_version": "1.0",
  "scenario_id": "scn_demo_01",
  "created_at": "2026-02-06T00:00:00Z",
  "language": "ko",
  "raw_text": "앞으로 3개월 안에 금리 인하가 나오고 성장주가 강세일 것 같아.",
  "source": "user",
  "context": { "asset_class_focus": ["equity"], "regions_focus": ["US"], "time_now": "2026-02-06" },
  "hypotheses": [
    {
      "id": "H1",
      "claim": "2026-02~2026-05 사이 정책금리 인하(또는 인하 기대 강화)로 US 성장주가 상대강세를 보일 것이다.",
      "time_horizon": { "start": "2026-02-06", "end": "2026-05-31", "confidence": 0.6, "granularity": "week" },
      "scope": { "regions": ["US"], "asset_classes": ["equity", "rates"], "tickers": ["QQQ"], "style_factors": ["growth"] },
      "drivers": [
        { "type": "policy", "factor": "policy_rate", "direction": "down", "magnitude": "small", "confidence": 0.5, "notes": "인플레 둔화 기대/경기 둔화" }
      ],
      "expected_market_effects": [
        { "target": "equity", "segment": "ticker:QQQ", "direction": "up", "strength": 0.7, "channel": "discount_rate", "time_lag": "2w" }
      ],
      "assumptions": [{ "text": "인플레이션이 재가속하지 않는다", "criticality": "high" }],
      "uncertainties": [{ "text": "연준이 인하를 지연할 수 있다", "impact": "high" }],
      "evidence": [{ "type": "user_belief", "summary": "최근 물가 둔화 체감", "confidence": 0.3 }],
      "tags": ["macro", "rates", "growth"]
    }
  ],
  "alternatives": [
    {
      "id": "A1",
      "relation": "inverse",
      "claim": "인하가 지연되거나 긴축 장기화로 성장주가 약세로 전환될 수 있다.",
      "time_horizon": { "start": "2026-02-06", "end": "2026-05-31", "confidence": 0.4, "granularity": "week" },
      "expected_market_effects": [
        { "target": "equity", "segment": "ticker:QQQ", "direction": "down", "strength": 0.6, "channel": "risk_premium", "time_lag": "0w" }
      ]
    }
  ],
  "recommended_edits": [
    { "type": "quantify_magnitude", "before": "금리 인하", "after": "25bp 인하 또는 기대 강화", "reason": "전략/테스트 설계에 필요" }
  ],
  "quality_score": { "clarity": 0.7, "testability": 0.7, "internal_consistency": 0.8, "actionability": 0.7, "overall": 0.73 },
  "integration_plan": {
    "mode": "soft",
    "constraints": [{ "scope": "portfolio", "type": "FACTOR_TILT", "params": { "segment": "factor:growth", "tilt": "overweight", "max_factor_exposure": 0.6 }, "confidence": 0.5 }],
    "priority_weights": [{ "target": "selection", "rule": "rs_percentile", "weight_delta": 0.05 }],
    "stress_tests": [{ "name": "scenario_inverse", "based_on": "A1", "required": true }, { "name": "cost_shock", "params": { "slippage_bps": 2.0 }, "required": true }]
  },
  "trace": { "parser_version": "scenario_parser_v0", "normalizer_version": "scenario_norm_v0", "edit_rules_version": "scenario_edits_v0" }
}
```

## 17.2 예시 B — 섹터 이벤트(공급망 차질 → 반도체 변동성 상승)

(구조만 참고, 생략 가능)

---

## 18) 다음 문서 연결(Traceability)

- 자연어 → Scenario JSON 변환기는 `MOD-470 Scenario Parser`에서 구현
    
- 품질 점수/수정 제안 규칙은 `MOD-471`로 분리
    
- 시나리오를 전략에 반영하는 규칙은 `MOD-472`에서 `integration_plan` 생성으로 구현
    
- 시나리오와 전략 DSL 연결은 `SCHEMA-330 Constraints` 및 `SCHEMA-300 portfolio_constraints/execution`과 연동
    

---
