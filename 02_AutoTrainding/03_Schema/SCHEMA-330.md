## Constraints Schema Specification (제약 스키마)

- **문서 ID**: SCHEMA-330
    
- **제목**: Constraints JSON Specification
    
- **버전**: 1.0
    
- **목표**: 실패 분석/사용자 정책/베이스라인 규칙을 **기계가 적용 가능한 제약(Constraints)** 으로 표현하여, 전략 생성·포트폴리오·체결·데이터 품질 단계에서 동일하게 사용한다.
    
- **범위**: Generator/Backtester/Execution/Portfolio/Risk/DataQuality에 적용되는 모든 제약.
    
- **비범위**: 제약을 “어떻게 찾아내는가”(Analyzer 로직)는 MOD-450에서 정의.
    

---

## 1) 설계 원칙

1. **적용 지점 명확화**: 제약은 반드시 `scope`를 갖고, 어디에 적용되는지 명확해야 한다.
    
2. **기계적 적용 가능**: 자연어 금지. `type + params`로 해석 가능해야 한다.
    
3. **추적성**: 어떤 실패/규칙에서 파생되었는지 기록(`derived_from`).
    
4. **충돌 처리 가능**: 우선순위/정책을 포함하여 충돌을 해결할 수 있어야 한다.
    
5. **진화 가능**: 활성/만료/버전 관리가 가능해야 한다(제약 과잉 방지).
    

---

## 2) 최상위 구조(Constraint Set)

제약은 보통 “세트(목록)”로 운용하며, iteration마다 스냅샷을 남긴다.

```json
{
  "schema_version": "1.0",
  "constraint_set_id": "uuid-or-hash",
  "created_at": "2026-02-06T00:00:00Z",
  "source": "baseline|analyzer|user|system",
  "status": "ACTIVE",
  "priority_policy": "FAILSAFE_FIRST",
  "constraints": []
}
```

### 2.1 필수 필드

- `schema_version`, `source`, `status`, `constraints`
    

---

## 3) 단일 Constraint 오브젝트 구조

```json
{
  "id": "uuid",
  "status": "ACTIVE|INACTIVE|DEPRECATED",
  "scope": "generator|portfolio|execution|risk|data",
  "type": "CONSTRAINT_TYPE",
  "params": {},
  "priority": 50,
  "severity": "INFO|WARN|ERROR|CRITICAL",
  "applies_to": {},
  "derived_from": {},
  "notes": "optional"
}
```

### 3.1 필수 필드

- `id`, `status`, `scope`, `type`, `params`, `priority`
    

### 3.2 권장 필드

- `severity` (운영 안전성 관련은 높게)
    
- `applies_to` (유니버스/섹터/전략 패밀리 등 타겟팅)
    
- `derived_from` (실패/시나리오/사용자 입력 링크)
    

---

## 4) scope 정의(적용 지점)

- `generator`: 전략 후보 생성 단계(템플릿/파라미터/복잡도/지표 사용 제한)
    
- `portfolio`: 포트폴리오 구성(종목수/섹터/상관/노출/레버리지)
    
- `execution`: 주문/체결 품질(ADV/스프레드/지연/주문 타입)
    
- `risk`: 스탑/트레일/손실 한도 등 리스크 규칙
    
- `data`: 데이터 품질/룩어헤드/캘린더/결측 정책
    

---

## 5) type 카탈로그(표준 제약 타입)

아래 타입들은 “기본 세트”로 정의한다. 신규 타입은 버전업으로 추가.

---

# 5A) generator scope 제약

## 5A-1) `BAN_TEMPLATE`

특정 템플릿(또는 룰 패턴)을 금지

```json
{
  "scope": "generator",
  "type": "BAN_TEMPLATE",
  "params": { "template_id": "osc_mean_reversion_01", "reason": "overfit" }
}
```

## 5A-2) `ALLOW_FEATURES_ONLY`

허용 지표(피처) 화이트리스트

```json
{
  "scope": "generator",
  "type": "ALLOW_FEATURES_ONLY",
  "params": { "features": ["rsi_14", "stoch_k_14_3_3", "ema_21"] }
}
```

## 5A-3) `BAN_FEATURES`

금지 지표 블랙리스트

```json
{
  "scope": "generator",
  "type": "BAN_FEATURES",
  "params": { "features": ["future_return_1d"] }
}
```

## 5A-4) `LIMIT_PARAM_RANGE`

파라미터 범위 제한(과최적화/민감도 실패 대응)

```json
{
  "scope": "generator",
  "type": "LIMIT_PARAM_RANGE",
  "params": {
    "feature": "rsi_14",
    "param": "threshold_low",
    "min": 20,
    "max": 35
  }
}
```

> `feature/param`은 “Generator 내부 파라미터명” 표준이 필요(= MOD-411).

## 5A-5) `MAX_COMPLEXITY`

AST 복잡도 상한

```json
{
  "scope": "generator",
  "type": "MAX_COMPLEXITY",
  "params": { "max_ast_depth": 4, "max_cmp_count": 8, "max_feature_count": 12 }
}
```

## 5A-6) `MIN_SIGNAL_DENSITY` / `MAX_SIGNAL_DENSITY`

거래가 너무 적거나(희소) 너무 많을 때(과매매) 신호 밀도 제한

```json
{
  "scope": "generator",
  "type": "MIN_SIGNAL_DENSITY",
  "params": { "min_trades_per_year": 10 }
}
```

```json
{
  "scope": "generator",
  "type": "MAX_SIGNAL_DENSITY",
  "params": { "max_trades_per_year": 300 }
}
```

---

# 5B) portfolio scope 제약

## 5B-1) `MAX_POSITIONS`

최대 보유 종목 수

```json
{
  "scope": "portfolio",
  "type": "MAX_POSITIONS",
  "params": { "value": 20 }
}
```

## 5B-2) `MAX_POSITION_WEIGHT`

단일 종목 최대 비중

```json
{ "scope": "portfolio", "type": "MAX_POSITION_WEIGHT", "params": { "value": 0.10 } }
```

## 5B-3) `SECTOR_CAP`

섹터 비중 상한

```json
{
  "scope": "portfolio",
  "type": "SECTOR_CAP",
  "params": { "max_weight": 0.30 }
}
```

## 5B-4) `SECTOR_TILT` (시나리오 soft 반영)

특정 섹터/테마를 overweight/underweight (단, cap 함께 쓰는 것을 권장)

```json
{
  "scope": "portfolio",
  "type": "SECTOR_TILT",
  "params": { "segment": "sector:Semiconductors", "tilt": "overweight", "tilt_strength": 0.2, "max_weight": 0.35 }
}
```

## 5B-5) `MAX_CORR_TO_PORTFOLIO`

포트폴리오와 상관 상한

```json
{
  "scope": "portfolio",
  "type": "MAX_CORR_TO_PORTFOLIO",
  "params": { "value": 0.85, "window_bars": 252 }
}
```

## 5B-6) `LEVERAGE_LIMIT`

레버리지 제한

```json
{
  "scope": "portfolio",
  "type": "LEVERAGE_LIMIT",
  "params": { "max_gross": 1.0, "max_net": 1.0 }
}
```

---

# 5C) execution scope 제약

## 5C-1) `ADV_PARTICIPATION_CAP`

ADV 대비 주문비중 제한

```json
{
  "scope": "execution",
  "type": "ADV_PARTICIPATION_CAP",
  "params": { "max_participation": 0.01, "adv_window_bars": 60 }
}
```

## 5C-2) `SPREAD_BPS_CAP`

스프레드 상한(체결 품질)

```json
{ "scope": "execution", "type": "SPREAD_BPS_CAP", "params": { "max_spread_bps": 10 } }
```

## 5C-3) `ORDER_TYPE_ALLOWLIST`

주문 타입 허용 목록(라이브 안전)

```json
{
  "scope": "execution",
  "type": "ORDER_TYPE_ALLOWLIST",
  "params": { "allowed": ["MKT", "LMT"] }
}
```

## 5C-4) `LATENCY_GUARD`

지연이 커지면 거래 중단/축소

```json
{
  "scope": "execution",
  "type": "LATENCY_GUARD",
  "params": { "max_latency_ms": 5000, "action": "halt|reduce_size" }
}
```

---

# 5D) risk scope 제약

## 5D-1) `MAX_DAILY_LOSS_PCT`

일 손실 한도(자동 중단)

```json
{ "scope": "risk", "type": "MAX_DAILY_LOSS_PCT", "params": { "value": 0.03 } }
```

## 5D-2) `MAX_DRAWDOWN_PCT`

최대 낙폭 한도

```json
{ "scope": "risk", "type": "MAX_DRAWDOWN_PCT", "params": { "value": 0.15 } }
```

## 5D-3) `STOP_DISTANCE_MIN_MAX`

스탑 거리(ADR/ATR 대비) 최소/최대 규정

```json
{
  "scope": "risk",
  "type": "STOP_DISTANCE_MIN_MAX",
  "params": { "min_atr_mult": 1.5, "max_atr_mult": 6.0 }
}
```

---

# 5E) data scope 제약

## 5E-1) `NO_LOOKAHEAD`

룩어헤드 방지 정책(필수)

```json
{
  "scope": "data",
  "type": "NO_LOOKAHEAD",
  "params": { "execution_timing": "next_bar", "disallow_same_bar_fill": true }
}
```

## 5E-2) `NAN_POLICY`

결측 처리 강제(전략 DSL과 일치해야 함)

```json
{
  "scope": "data",
  "type": "NAN_POLICY",
  "params": { "on_nan_in_condition": "DISALLOW_TRADE", "reason_code": "DATA_MISSING" }
}
```

## 5E-3) `CALENDAR_POLICY`

캘린더/타임존 정합성

```json
{
  "scope": "data",
  "type": "CALENDAR_POLICY",
  "params": { "timezone": "America/New_York", "trading_calendar": "NYSE" }
}
```

---

## 6) applies_to (타겟팅)

제약이 어디에 적용되는지 세부 지정.

```json
"applies_to": {
  "asset_class": ["equity"],
  "regions": ["US"],
  "symbols": ["AAPL", "MSFT"],
  "sectors": ["Semiconductors"],
  "strategy_families": ["trend_follow"],
  "template_ids": ["osc_mean_reversion_01"]
}
```

- 비어있으면 “전역 적용” 의미
    

---

## 7) derived_from (추적성)

```json
"derived_from": {
  "type": "failure|scenario|user|baseline",
  "id": "failure_id_or_scenario_id",
  "evidence_refs": [
    { "kind": "metric", "key": "param_sensitivity_score", "value": 0.2 },
    { "kind": "failure_tag", "tag": "Threshold_Overfit" }
  ]
}
```

---

## 8) 충돌/우선순위 정책(priority_policy)

Constraint Set 레벨 정책(권장)

- `FAILSAFE_FIRST`: risk/execution/data 관련 제약을 최우선 적용
    
- `MOST_RECENT_WINS`: 동일 scope/type 충돌 시 최신 제약 우선
    
- `HIGHEST_PRIORITY_WINS`: `priority` 값이 큰 것이 우선
    

각 constraint에는 `priority (0~100)`를 부여.

---

## 9) Validator 요구사항(필수 체크)

1. `constraints` 배열 내 `id`는 유일
    
2. `scope`는 허용값 중 하나
    
3. `type`은 카탈로그에 존재
    
4. `params`는 type별 필수 키를 모두 포함
    
5. `priority`는 0~100
    
6. risk/execution/data scope 제약이 비활성화(INACTIVE)인 경우 경고(운영 안전)
    

---

## 10) 예시: 실패 → 제약 생성(과최적화/과매매/체결 품질)

### 10.1 Threshold Overfit → 파라미터 범위 축소 + 복잡도 제한

```json
{
  "schema_version": "1.0",
  "constraint_set_id": "cs_demo_01",
  "created_at": "2026-02-06T00:00:00Z",
  "source": "analyzer",
  "status": "ACTIVE",
  "priority_policy": "FAILSAFE_FIRST",
  "constraints": [
    {
      "id": "c1",
      "status": "ACTIVE",
      "scope": "generator",
      "type": "LIMIT_PARAM_RANGE",
      "params": { "feature": "rsi_14", "param": "threshold_low", "min": 20, "max": 35 },
      "priority": 60,
      "severity": "WARN",
      "derived_from": { "type": "failure", "id": "f_123", "evidence_refs": [{ "kind": "failure_tag", "tag": "Threshold_Overfit" }] }
    },
    {
      "id": "c2",
      "status": "ACTIVE",
      "scope": "generator",
      "type": "MAX_COMPLEXITY",
      "params": { "max_ast_depth": 4, "max_cmp_count": 6, "max_feature_count": 10 },
      "priority": 70,
      "severity": "WARN",
      "derived_from": { "type": "failure", "id": "f_123" }
    }
  ]
}
```

### 10.2 Execution Failure → 스프레드/ADV 제한 강화

```json
{
  "id": "c_exec_1",
  "status": "ACTIVE",
  "scope": "execution",
  "type": "SPREAD_BPS_CAP",
  "params": { "max_spread_bps": 6 },
  "priority": 90,
  "severity": "ERROR",
  "derived_from": { "type": "failure", "id": "f_exec_9" }
}
```

---

## 11) 다음 문서 연결(Traceability)

- 실패→제약 변환 로직: **MOD-450 Constraint Writer**
    
- 제약 적용 지점(Generator/Portfolio/Execution): **MOD-451 Constraint Applicator**
    
- 제약 라이프사이클(만료/회귀): **MOD-452 Constraint Lifecycle**
    
- 시나리오 반영 제약 생성: **SCHEMA-320 integration_plan.constraints**
    
- 저장: DB `constraints.constraint_json` 및 iteration snapshot
    

---
