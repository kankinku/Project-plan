# Strategy DSL Specification

- **문서 ID**: SCHEMA-300
    
- **제목**: Strategy DSL(JSON) Specification
    
- **버전**: 1.0
    
- **목표**: 전략을 _생성/백테스트/평가/로그/재현/설명_ 가능하도록 **단일 JSON 스펙(DSL)**로 정의한다.
    
- **범위**: 백테스트/페이퍼/라이브 모두 동일 DSL 사용.
    
- **비범위**: 브로커별 주문 API 세부(별도 MOD-480 문서).
    

---

## 1. 설계 원칙

1. **재현성**: 동일 DSL + 동일 데이터/설정 버전이면 결과 리플레이 가능해야 한다.
    
2. **관측 가능성**: “왜 매수/매도/미실행”이었는지 reason code + 근거값(features)을 남길 수 있어야 한다.
    
3. **모듈화**: 레짐/유동성/선정/진입/청산/리스크/실행/비용을 분리한다.
    
4. **안전성**: 룩어헤드/NaN/타입 오류를 스키마 레벨에서 최대한 차단한다.
    
5. **확장성**: AST(조건식)와 rule template, 시나리오 반영을 자연스럽게 붙일 수 있어야 한다.
    

---

## 2. 최상위 구조(Top-level)

전략 DSL은 아래 최상위 키를 가진다.

```json
{
  "schema_version": "1.0",
  "strategy_id": "hash-or-uuid",
  "name": "optional human name",
  "family": "osc_mean_reversion|trend_follow|...",
  "metadata": {},
  "universe": {},
  "features": {},
  "modules": {},
  "risk": {},
  "portfolio_constraints": {},
  "execution": {},
  "cost_model": {},
  "logging": {}
}
```

### 2.1 필수 필드

- `schema_version`
    
- `universe`
    
- `features`
    
- `modules` (최소 entry/exit 포함)
    
- `risk`
    
- `execution`
    
- `cost_model`
    
- `logging`
    

---

## 3. metadata

전략 생성/재현을 위한 메타 정보.

```json
"metadata": {
  "created_by": "generator|user|llm",
  "generator_version": "gen_v0",
  "created_at": "2026-02-06T00:00:00Z",
  "notes": "optional",
  "tags": ["rsi", "stoch", "adx"],
  "baseline_ref": "baseline_v1",
  "data_requirements": {
    "bar_interval": "1d",
    "lookback_bars": 200
  }
}
```

---

## 4. universe

### 4.1 유니버스 정의

```json
"universe": {
  "asset_class": "equity|etf|crypto|futures",
  "symbols": ["AAPL", "MSFT"],
  "or": {
    "universe_source": "s&p500|custom_list|dynamic",
    "universe_source_params": {}
  },
  "tradable_filter": {
    "min_avg_vol_60": 1000000,
    "min_dollar_vol_week": 5000000,
    "max_spread_bps": 10,
    "exclude_halted": true,
    "exclude_delist_risk": true
  },
  "ranking": {
    "method": "rs_percentile",
    "benchmark_symbol": "SPY",
    "window_bars": 252,
    "min_rs_percentile": 90
  }
}
```

### 4.2 규칙

- `symbols` 직접 지정 또는 `universe_source` 둘 중 하나는 필수
    
- `tradable_filter`는 실거래/현실성 품질에 직결 → 기본 포함 권장
    
- `ranking`은 선택이지만, “RS>90 같은 조건”이 필요하면 여기서 고정
    

---

## 5. features (지표/피처 정의)

전략에서 사용하는 모든 지표는 `features`에 명시한다.  
키는 **표준 이름(소문자_snake)** + 파라미터를 암시할 수 있게 한다.

```json
"features": {
  "rsi_14": { "type": "rsi", "inputs": ["close"], "params": { "period": 14 } },
  "stoch_k_14_3_3": { "type": "stoch_k", "inputs": ["high", "low", "close"], "params": { "k": 14, "d": 3, "smooth": 3 } },
  "adx_14": { "type": "adx", "inputs": ["high", "low", "close"], "params": { "period": 14 } },
  "di_plus_14": { "type": "di_plus", "inputs": ["high", "low", "close"], "params": { "period": 14 } },
  "di_minus_14": { "type": "di_minus", "inputs": ["high", "low", "close"], "params": { "period": 14 } },
  "ema_8": { "type": "ema", "inputs": ["close"], "params": { "period": 8 } },
  "ema_21": { "type": "ema", "inputs": ["close"], "params": { "period": 21 } },
  "atr_pct_14": { "type": "atr_pct", "inputs": ["high", "low", "close"], "params": { "period": 14 } }
}
```

### 5.1 NaN 정책(필수)

```json
"metadata": {
  "nan_policy": {
    "on_nan_in_condition": "DISALLOW_TRADE", 
    "reason_code": "DATA_MISSING"
  }
}
```

---

## 6. modules

전략의 의사결정 모듈 집합. 최소 `entry`/`exit`는 필수.

### 6.1 regime(선택)

```json
"modules": {
  "regime": {
    "enabled": true,
    "method": "benchmark_trend_vol",
    "params": { "benchmark": "SPY", "ma_period": 200, "vix_threshold": 25 }
  },
  ...
}
```

### 6.2 entry (필수, AST 조건식)

```json
"modules": {
  "entry": {
    "enabled": true,
    "filter": { "ref": "AST_FILTER_1" },
    "condition": { "ref": "AST_ENTRY_1" },
    "cooldown_bars": 0,
    "score_model": {
      "enabled": true,
      "weights": { "filter": 20, "trend": 20, "strength": 20, "osc": 40 }
    },
    "reason_code_map": "default_entry_reason_map_v1"
  }
}
```

### 6.3 exit (필수)

```json
"modules": {
  "exit": {
    "enabled": true,
    "condition": { "ref": "AST_EXIT_1" },
    "time_stop": { "enabled": false, "max_holding_bars": 60 },
    "reason_code_map": "default_exit_reason_map_v1"
  }
}
```

### 6.4 selection(선택: RS/분산 제약과 결합 권장)

```json
"modules": {
  "selection": {
    "enabled": true,
    "top_n": 20,
    "min_rs_percentile": 90
  }
}
```

### 6.5 AST 정의 위치(표준)

DSL 내부에 `conditions` 블록을 둬서 모든 AST를 한 곳에 모은다.

```json
"modules": {
  "...": {}
},
"conditions": {
  "AST_FILTER_1": { "... AST ..." },
  "AST_ENTRY_1": { "... AST ..." },
  "AST_EXIT_1": { "... AST ..." }
}
```

---

## 7. risk

리스크는 “트레이드 보호” + “포트폴리오 보호”로 나눈다.

```json
"risk": {
  "trade_risk": {
    "hard_stop": { "type": "pct", "value": 0.06 },
    "trailing_stop": { "type": "atr", "mult": 3.0 },
    "take_profit": { "enabled": false, "levels": [] }
  },
  "portfolio_risk": {
    "max_daily_loss_pct": 0.03,
    "max_drawdown_pct": 0.15,
    "halt_on_n_exec_failures": 5
  }
}
```

---

## 8. portfolio_constraints

섹터 쏠림/상관/종목수 제한 등.

```json
"portfolio_constraints": {
  "max_positions": 20,
  "max_position_weight": 0.10,
  "max_sector_weight": 0.30,
  "max_corr_to_portfolio": 0.85,
  "leverage": { "max_gross": 1.0, "max_net": 1.0 }
}
```

---

## 9. execution

체결과 리밸런싱 규칙.

```json
"execution": {
  "mode": "backtest|paper|live",
  "rebalance": { "frequency": "daily|weekly|monthly", "day_of_week": "FRI" },
  "order": {
    "default_order_type": "MKT",
    "time_in_force": "DAY",
    "max_adv_participation": 0.01,
    "reject_if_spread_bps_gt": 10
  },
  "fill_policy": {
    "use_next_bar": true,
    "slippage_handling": "model_based"
  }
}
```

---

## 10. cost_model

```json
"cost_model": {
  "fees": { "type": "bps", "value_bps": 1.0 },
  "spread": { "type": "bps", "value_bps": 2.0 },
  "slippage": { "type": "bps", "value_bps": 3.0 }
}
```

---

## 11. logging

관측 가능성을 DSL에서 강제한다.

```json
"logging": {
  "decision_log": { "enabled": true, "level": "BAR", "include_features": true },
  "order_fill_log": { "enabled": true },
  "trade_summary": { "enabled": true },
  "reason_codes": { "enabled": true, "standard": "reason_codes_v1" }
}
```

---

## 12. 검증 규칙(Validator Requirements)

전략 DSL은 실행 전에 반드시 아래를 통과해야 한다.

### 12.1 스키마/참조 무결성

- `modules.entry.condition.ref`는 `conditions`에 존재해야 함
    
- `features` 키는 유일해야 함
    
- `conditions`에서 참조하는 feature key는 `features`에 존재해야 함
    

### 12.2 안전 규칙

- `metadata.nan_policy.on_nan_in_condition = DISALLOW_TRADE` 기본 권장
    
- `execution.fill_policy.use_next_bar = true` (종가 신호 룩어헤드 방지)
    
- `cost_model`의 모든 값은 0 이상
    

### 12.3 복잡도 제한(초기 기본값)

- AST 최대 depth: 4
    
- CMP 최대 개수: 8  
    (초기엔 단순 전략으로 안정화 후 확장)
    

---

# 13. 예시 3개

## 13.1 예시 A — 오실레이터 조건식(요청 반영: Stoch >= X AND RSI <= Y)

```json
{
  "schema_version": "1.0",
  "strategy_id": "demo_osc_01",
  "name": "Oscillator Mean Reversion",
  "family": "osc_mean_reversion",
  "metadata": {
    "created_by": "user",
    "data_requirements": { "bar_interval": "1d", "lookback_bars": 200 },
    "nan_policy": { "on_nan_in_condition": "DISALLOW_TRADE", "reason_code": "DATA_MISSING" }
  },
  "universe": {
    "asset_class": "equity",
    "universe_source": "s&p500",
    "tradable_filter": { "min_avg_vol_60": 1000000, "min_dollar_vol_week": 5000000, "max_spread_bps": 10 }
  },
  "features": {
    "rsi_14": { "type": "rsi", "inputs": ["close"], "params": { "period": 14 } },
    "stoch_k_14_3_3": { "type": "stoch_k", "inputs": ["high","low","close"], "params": { "k": 14, "d": 3, "smooth": 3 } }
  },
  "conditions": {
    "AST_ENTRY_1": {
      "type": "AND",
      "children": [
        { "type": "CMP", "left": "stoch_k_14_3_3", "op": ">=", "right": 80 },
        { "type": "CMP", "left": "rsi_14", "op": "<=", "right": 30 }
      ]
    },
    "AST_EXIT_1": {
      "type": "OR",
      "children": [
        { "type": "CMP", "left": "rsi_14", "op": ">=", "right": 55 },
        { "type": "CMP", "left": "stoch_k_14_3_3", "op": "<=", "right": 20 }
      ]
    }
  },
  "modules": {
    "entry": { "enabled": true, "condition": { "ref": "AST_ENTRY_1" }, "cooldown_bars": 0, "reason_code_map": "default_entry_reason_map_v1" },
    "exit": { "enabled": true, "condition": { "ref": "AST_EXIT_1" }, "reason_code_map": "default_exit_reason_map_v1" }
  },
  "risk": { "trade_risk": { "hard_stop": { "type": "pct", "value": 0.06 }, "trailing_stop": { "type": "atr", "mult": 3.0 } }, "portfolio_risk": { "max_daily_loss_pct": 0.03, "max_drawdown_pct": 0.15 } },
  "portfolio_constraints": { "max_positions": 10, "max_position_weight": 0.12, "max_sector_weight": 0.35, "max_corr_to_portfolio": 0.90, "leverage": { "max_gross": 1.0, "max_net": 1.0 } },
  "execution": { "mode": "backtest", "rebalance": { "frequency": "daily" }, "order": { "default_order_type": "MKT", "max_adv_participation": 0.01, "reject_if_spread_bps_gt": 10 }, "fill_policy": { "use_next_bar": true, "slippage_handling": "model_based" } },
  "cost_model": { "fees": { "type": "bps", "value_bps": 1.0 }, "spread": { "type": "bps", "value_bps": 2.0 }, "slippage": { "type": "bps", "value_bps": 3.0 } },
  "logging": { "decision_log": { "enabled": true, "level": "BAR", "include_features": true }, "order_fill_log": { "enabled": true }, "trade_summary": { "enabled": true }, "reason_codes": { "enabled": true, "standard": "reason_codes_v1" } }
}
```

## 13.2 예시 B — 추세추종(EMA + ADX + DI)

> (SCHEMA-310에서 AST 타입/정규화 규칙을 더 엄밀히 정의)

```json
{
  "schema_version": "1.0",
  "strategy_id": "demo_trend_01",
  "family": "trend_follow",
  "universe": { "asset_class": "equity", "universe_source": "s&p500", "tradable_filter": { "min_avg_vol_60": 1000000, "max_spread_bps": 10 } },
  "features": {
    "adx_14": { "type": "adx", "inputs": ["high","low","close"], "params": { "period": 14 } },
    "di_plus_14": { "type": "di_plus", "inputs": ["high","low","close"], "params": { "period": 14 } },
    "di_minus_14": { "type": "di_minus", "inputs": ["high","low","close"], "params": { "period": 14 } },
    "ema_8": { "type": "ema", "inputs": ["close"], "params": { "period": 8 } },
    "ema_21": { "type": "ema", "inputs": ["close"], "params": { "period": 21 } },
    "ema_55": { "type": "ema", "inputs": ["close"], "params": { "period": 55 } }
  },
  "conditions": {
    "AST_ENTRY_1": {
      "type": "AND",
      "children": [
        { "type": "CMP", "left": "adx_14", "op": ">=", "right": 20 },
        { "type": "CMP", "left": "di_plus_14", "op": ">", "right": "di_minus_14" },
        { "type": "AND", "children": [
          { "type": "CMP", "left": "ema_8", "op": ">", "right": "ema_21" },
          { "type": "CMP", "left": "ema_21", "op": ">", "right": "ema_55" }
        ]}
      ]
    },
    "AST_EXIT_1": {
      "type": "OR",
      "children": [
        { "type": "CMP", "left": "di_plus_14", "op": "<", "right": "di_minus_14" },
        { "type": "CMP", "left": "ema_8", "op": "<", "right": "ema_21" }
      ]
    }
  },
  "modules": { "entry": { "enabled": true, "condition": { "ref": "AST_ENTRY_1" } }, "exit": { "enabled": true, "condition": { "ref": "AST_EXIT_1" } } },
  "risk": { "trade_risk": { "hard_stop": { "type": "pct", "value": 0.08 }, "trailing_stop": { "type": "atr", "mult": 3.0 } }, "portfolio_risk": { "max_daily_loss_pct": 0.03, "max_drawdown_pct": 0.15 } },
  "portfolio_constraints": { "max_positions": 20, "max_position_weight": 0.10, "max_sector_weight": 0.30, "max_corr_to_portfolio": 0.85, "leverage": { "max_gross": 1.0, "max_net": 1.0 } },
  "execution": { "mode": "backtest", "rebalance": { "frequency": "daily" }, "order": { "default_order_type": "MKT", "max_adv_participation": 0.01, "reject_if_spread_bps_gt": 10 }, "fill_policy": { "use_next_bar": true } },
  "cost_model": { "fees": { "type": "bps", "value_bps": 1.0 }, "spread": { "type": "bps", "value_bps": 2.0 }, "slippage": { "type": "bps", "value_bps": 3.0 } },
  "logging": { "decision_log": { "enabled": true, "level": "BAR", "include_features": true }, "order_fill_log": { "enabled": true }, "trade_summary": { "enabled": true }, "reason_codes": { "enabled": true, "standard": "reason_codes_v1" } }
}
```

## 13.3 예시 C — 레짐 필터 포함(리스크 오프 시 거래 금지)

```json
{
  "schema_version": "1.0",
  "strategy_id": "demo_regime_filter_01",
  "family": "trend_follow",
  "metadata": { "nan_policy": { "on_nan_in_condition": "DISALLOW_TRADE", "reason_code": "DATA_MISSING" } },
  "universe": { "asset_class": "equity", "universe_source": "s&p500", "tradable_filter": { "min_avg_vol_60": 1000000, "max_spread_bps": 10 } },
  "features": { "rsi_14": { "type": "rsi", "inputs": ["close"], "params": { "period": 14 } } },
  "conditions": {
    "AST_FILTER_1": { "type": "CMP", "left": "regime_state", "op": "==", "right": "RISK_ON" },
    "AST_ENTRY_1": { "type": "CMP", "left": "rsi_14", "op": "<=", "right": 35 },
    "AST_EXIT_1": { "type": "CMP", "left": "rsi_14", "op": ">=", "right": 55 }
  },
  "modules": {
    "regime": { "enabled": true, "method": "benchmark_trend_vol", "params": { "benchmark": "SPY", "ma_period": 200, "vix_threshold": 25 } },
    "entry": { "enabled": true, "filter": { "ref": "AST_FILTER_1" }, "condition": { "ref": "AST_ENTRY_1" } },
    "exit": { "enabled": true, "condition": { "ref": "AST_EXIT_1" } }
  },
  "risk": { "trade_risk": { "hard_stop": { "type": "pct", "value": 0.06 }, "trailing_stop": { "type": "atr", "mult": 3.0 } }, "portfolio_risk": { "max_daily_loss_pct": 0.03, "max_drawdown_pct": 0.15 } },
  "portfolio_constraints": { "max_positions": 10, "max_position_weight": 0.12, "max_sector_weight": 0.35, "max_corr_to_portfolio": 0.90, "leverage": { "max_gross": 1.0, "max_net": 1.0 } },
  "execution": { "mode": "backtest", "rebalance": { "frequency": "daily" }, "order": { "default_order_type": "MKT", "max_adv_participation": 0.01, "reject_if_spread_bps_gt": 10 }, "fill_policy": { "use_next_bar": true } },
  "cost_model": { "fees": { "type": "bps", "value_bps": 1.0 }, "spread": { "type": "bps", "value_bps": 2.0 }, "slippage": { "type": "bps", "value_bps": 3.0 } },
  "logging": { "decision_log": { "enabled": true, "level": "BAR", "include_features": true }, "order_fill_log": { "enabled": true }, "trade_summary": { "enabled": true }, "reason_codes": { "enabled": true, "standard": "reason_codes_v1" } }
}
```

---

## 14. 다음 문서 연결(Traceability)

- AST 노드 타입/정규화/타입 체크 규칙은 **SCHEMA-310**에서 확정한다.
    
- Reason code 표준/사전은 **DOC-003 / configs/reason_codes.yaml**로 관리한다.
    
- 로그 저장 규격은 **OBS-500/510/520**을 따른다.