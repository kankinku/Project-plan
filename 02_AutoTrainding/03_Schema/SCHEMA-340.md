## Metrics Schema Specification (표준 지표 스키마)

- **문서 ID**: SCHEMA-340
    
- **제목**: Metrics Schema Specification (Names, Units, Aggregation, Calculation Contracts)
    
- **버전**: 1.0
    
- **목표**: 백테스트/라이브/리플레이 전 구간에서 **성과·위험·거래품질·신뢰도** 지표를 동일한 이름/단위/계산 규칙으로 저장·비교·검증 가능하게 한다.
    
- **범위**: `strategies.metrics_json`, 리포트, 실패 분석(evidence), 제약 생성에서 사용하는 모든 지표.
    
- **비범위**: 각 엔진(vectorbt/backtrader/lean) 별 세부 구현(별도 MOD-430).
    

---

## 1) 설계 원칙

1. **일관된 네이밍**: 지표 이름은 고정된 표준 키로 저장한다.
    
2. **단위/연율화 명시**: `%`인지 비율인지, 연율화 여부를 메타로 함께 둔다.
    
3. **레벨 구분**: portfolio-level / symbol-level / trade-level / regime-slice-level을 명확히 분리한다.
    
4. **재현 가능 계약(Contract)**: 계산에 쓰인 입력(수익률 시계열, 비용 포함 여부 등)을 메타로 기록한다.
    
5. **확장 가능**: 새 지표 추가 시 기존 시스템과 충돌 없이 버전 관리한다.
    

---

## 2) 최상위 구조(Top-level Metrics Object)

전략 결과에는 `metrics_json`(전체)과 선택적으로 `score_breakdown_json`이 저장된다.

```json
{
  "schema_version": "1.0",
  "as_of": "2026-02-06T00:00:00Z",
  "bar_interval": "1d",
  "currency": "USD",
  "period": { "start": "2015-01-01", "end": "2025-12-31" },
  "engine": { "name": "vectorbt", "version": "x.y.z" },
  "calc_contract": {
    "returns_type": "log|simple",
    "include_fees": true,
    "include_slippage": true,
    "include_spread": true,
    "execution_timing": "next_bar|same_bar_disallowed",
    "risk_free_rate_annual": 0.02,
    "annualization_factor": 252
  },
  "metrics": {
    "portfolio": {},
    "symbols": {},
    "trades": {},
    "slices": {}
  }
}
```

### 2.1 필수 필드

- `schema_version`, `bar_interval`, `currency`, `period`, `calc_contract`, `metrics.portfolio`
    

---

## 3) 네이밍 규칙(Naming Convention)

### 3.1 공통 규칙

- 키는 `snake_case`
    
- 파생/변형은 suffix로 구분:
    
    - `_gross` / `_net`
        
    - `_annual` / `_monthly`
        
    - `_mean` / `_median` / `_p95`
        
- “비율”은 기본적으로 0~1 실수로 저장하고, `%` 표기는 UI에서 처리(단, 명확히 `*_pct` 키로 저장 가능)
    

### 3.2 레벨별 키 위치

- 포트폴리오 레벨: `metrics.portfolio.<metric_key>`
    
- 심볼 레벨: `metrics.symbols.<symbol>.<metric_key>`
    
- 트레이드 레벨 집계: `metrics.trades.<metric_key>`
    
- 구간/국면 슬라이스: `metrics.slices.<slice_name>.<metric_key>`
    

---

## 4) calc_contract (계산 계약/입력 조건)

지표를 해석하려면 “무엇을 포함했는지”가 필수.

```json
"calc_contract": {
  "returns_type": "simple",
  "include_fees": true,
  "include_slippage": true,
  "include_spread": true,
  "fees_model_version": "fees_v1",
  "slippage_model_version": "slip_v1",
  "spread_model_version": "spread_v1",
  "execution_timing": "next_bar",
  "annualization_factor": 252,
  "risk_free_rate_annual": 0.02,
  "drawdown_method": "peak_to_trough"
}
```

### 4.1 필수 규칙

- `annualization_factor`는 bar_interval에 맞게 고정(예: 1d=252, 1h=~1638 등은 정책 필요)
    
- `execution_timing`은 룩어헤드 방지를 위해 기본 `next_bar`
    

---

## 5) portfolio 메트릭 표준(핵심)

아래 키들은 **최소 필수 세트(MVP)** 와 **권장 세트(V1)** 로 나눈다.

## 5.1 MVP 필수(Portfolio)

```json
"metrics": {
  "portfolio": {
    "return_total_net": 0.52,
    "cagr_net": 0.08,
    "volatility_annual_net": 0.16,
    "max_drawdown_net": -0.22,

    "sharpe_net": 0.65,
    "sortino_net": 0.90,
    "calmar_net": 0.36,

    "win_rate_period": 0.54,
    "profit_factor": 1.25,

    "turnover_annual": 3.4,
    "avg_exposure": 0.72,

    "costs_total": 0.07,
    "fees_total": 0.01,
    "slippage_total": 0.04,
    "spread_total": 0.02,

    "oos_return_total_net": 0.12,
    "oos_sharpe_net": 0.40,
    "oos_is_performance_ratio": 0.55
  }
}
```

### 정의(핵심)

- `return_total_net`: 기간 총수익(비용 포함)
    
- `cagr_net`: 연복리 수익률
    
- `volatility_annual_net`: 연율화 변동성
    
- `max_drawdown_net`: 최대낙폭(음수)
    
- `sharpe_net/sortino_net/calmar_net`: 표준 정의(연율화 포함)
    
- `turnover_annual`: 연간 매매회전율(정의는 MOD-430에서 고정)
    
- `avg_exposure`: 평균 투자 비중(현금 제외 총노출)
    
- 비용 4종: costs_total = fees_total + slippage_total + spread_total
    
- OOS 지표: OOS 구간에서 동일 지표(반드시 net)
    

---

## 5.2 V1 권장(Portfolio Robustness)

```json
"metrics": {
  "portfolio": {
    "skewness": -0.2,
    "kurtosis": 4.1,
    "tail_loss_p95": -0.03,
    "tail_gain_p95": 0.04,

    "drawdown_duration_max_bars": 120,
    "ulcer_index": 0.12,

    "hit_rate_trades": 0.48,
    "avg_trade_return_net": 0.012,
    "avg_trade_duration_bars": 14,

    "exposure_max": 0.98,
    "leverage_max_gross": 1.0,

    "slippage_p95": 0.0015,
    "spread_bps_mean": 3.2
  }
}
```

---

## 6) symbols 메트릭(심볼별)

심볼별 성능/리스크를 기록(포트폴리오 쏠림 분석에 필요).

```json
"metrics": {
  "symbols": {
    "AAPL": {
      "return_total_net": 0.08,
      "contribution_to_pnl_net": 0.12,
      "max_drawdown_net": -0.15,
      "turnover_annual": 2.1,
      "avg_exposure": 0.05
    },
    "MSFT": { "...": "..." }
  }
}
```

### 권장 키

- `contribution_to_pnl_net` (전체 PnL 중 기여도)
    
- `avg_exposure`, `max_exposure`
    
- `trade_count`
    

---

## 7) trades 메트릭(트레이드 단위 집계)

트레이드의 질을 정량화(Exit/Risk 실패 원인 분해에 중요).

```json
"metrics": {
  "trades": {
    "trade_count": 420,
    "win_rate_trades": 0.47,
    "profit_factor": 1.30,
    "expectancy_net": 0.0012,

    "avg_win_net": 0.018,
    "avg_loss_net": -0.011,
    "win_loss_ratio": 1.64,

    "mfe_mean": 0.025,
    "mae_mean": -0.014,
    "mfe_mae_ratio": 1.79,

    "holding_period_mean_bars": 12,
    "holding_period_median_bars": 7
  }
}
```

---

## 8) slices 메트릭(국면/구간별)

워크포워드/레짐별 성능을 저장해 신뢰도 평가에 사용.

### 8.1 표준 slice_name

- `is_train`, `is_validation`, `is_test`
    
- `regime_risk_on`, `regime_risk_off`
    
- `vol_high`, `vol_low`
    
- `market_trend_up`, `market_trend_down`
    

```json
"metrics": {
  "slices": {
    "is_test": {
      "return_total_net": 0.12,
      "sharpe_net": 0.40,
      "max_drawdown_net": -0.10,
      "trade_count": 80
    },
    "regime_risk_off": {
      "return_total_net": -0.03,
      "max_drawdown_net": -0.08
    }
  }
}
```

---

## 9) 신뢰도/강건성 지표(필수 권장)

조건식 전략(임계값 기반)은 과최적화 위험이 커서 아래 지표를 스키마에 표준으로 둔다.

### 9.1 OOS 유지율/비율

- `oos_is_performance_ratio`:
    
    - 예: `oos_return_total_net / is_return_total_net` 또는 스코어 비율(정의는 고정 필요)
        
- `oos_sharpe_net`, `oos_max_drawdown_net`
    

### 9.2 민감도(Threshold Sensitivity)

임계값/파라미터를 ±Δ 했을 때 성능 안정성.

```json
"metrics": {
  "portfolio": {
    "param_sensitivity_score": 0.62,
    "param_sensitivity_details": {
      "rsi_14_threshold_delta": 5,
      "stoch_k_delta": 5,
      "score_drop_mean": 0.12,
      "score_drop_p95": 0.25
    }
  }
}
```

---

## 10) 비용/체결 품질 지표(필수 권장)

실전 자동매매 기준에서 비용 분해는 필수.

```json
"metrics": {
  "portfolio": {
    "costs_total": 0.07,
    "fees_total": 0.01,
    "slippage_total": 0.04,
    "spread_total": 0.02,

    "slippage_mean": 0.0004,
    "slippage_p95": 0.0015,
    "fill_reject_rate": 0.002,
    "partial_fill_rate": 0.01
  }
}
```

---

## 11) Validator 요구사항(필수 체크)

1. `metrics.portfolio`는 존재해야 함
    
2. 수익/위험 핵심 키(MVP 필수) 누락 시 FAIL
    
3. `max_drawdown_*`는 0 이하(음수) 규칙 권장
    
4. 비용 합산 일관성: `costs_total ≈ fees_total + slippage_total + spread_total` (허용 오차 정책 필요)
    
5. OOS 키가 있으면 period 분할 정보(`period` 또는 slices의 train/test)가 함께 존재해야 함
    
6. `calc_contract` 필수 필드 누락 시 FAIL
    

---

## 12) 표준 키 목록(요약)

### Portfolio 핵심

- `return_total_net`, `cagr_net`, `volatility_annual_net`, `max_drawdown_net`
    
- `sharpe_net`, `sortino_net`, `calmar_net`
    
- `turnover_annual`, `avg_exposure`
    
- `costs_total`, `fees_total`, `slippage_total`, `spread_total`
    
- `oos_return_total_net`, `oos_sharpe_net`, `oos_is_performance_ratio`
    
- `param_sensitivity_score`
    

### Trades 핵심

- `trade_count`, `win_rate_trades`, `profit_factor`, `expectancy_net`
    
- `avg_win_net`, `avg_loss_net`, `holding_period_mean_bars`
    
- `mfe_mean`, `mae_mean`
    

---

## 13) 예시(전체 Metrics JSON 샘플)

```json
{
  "schema_version": "1.0",
  "as_of": "2026-02-06T00:00:00Z",
  "bar_interval": "1d",
  "currency": "USD",
  "period": { "start": "2015-01-01", "end": "2025-12-31" },
  "engine": { "name": "vectorbt", "version": "0.26.0" },
  "calc_contract": {
    "returns_type": "simple",
    "include_fees": true,
    "include_slippage": true,
    "include_spread": true,
    "execution_timing": "next_bar",
    "risk_free_rate_annual": 0.02,
    "annualization_factor": 252,
    "drawdown_method": "peak_to_trough"
  },
  "metrics": {
    "portfolio": {
      "return_total_net": 0.52,
      "cagr_net": 0.08,
      "volatility_annual_net": 0.16,
      "max_drawdown_net": -0.22,
      "sharpe_net": 0.65,
      "sortino_net": 0.90,
      "calmar_net": 0.36,
      "turnover_annual": 3.4,
      "avg_exposure": 0.72,
      "costs_total": 0.07,
      "fees_total": 0.01,
      "slippage_total": 0.04,
      "spread_total": 0.02,
      "oos_return_total_net": 0.12,
      "oos_sharpe_net": 0.40,
      "oos_is_performance_ratio": 0.55,
      "param_sensitivity_score": 0.62
    },
    "trades": {
      "trade_count": 420,
      "win_rate_trades": 0.47,
      "profit_factor": 1.30,
      "expectancy_net": 0.0012,
      "holding_period_mean_bars": 12,
      "mfe_mean": 0.025,
      "mae_mean": -0.014
    },
    "symbols": {},
    "slices": {
      "is_test": { "return_total_net": 0.12, "sharpe_net": 0.40, "max_drawdown_net": -0.10, "trade_count": 80 }
    }
  }
}
```

---

## 14) 다음 문서 연결(Traceability)

- 지표 계산/연율화/리샘플링 구현: **MOD-430 Metrics Calculator**
    
- 점수화(가중치/페널티/OOS/민감도): **MOD-431 Scorer**
    
- 실패 분석에서 evidence로 어떤 지표를 참조할지: **MOD-442 Evidence Builder**
    
- DB 저장 스키마: `strategies.metrics_json`, `strategy_metrics_timeseries` (DB 설계 문서)
    

---
