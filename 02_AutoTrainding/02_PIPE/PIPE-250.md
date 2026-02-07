## Scenario-to-Strategy Pipeline

- **문서 ID**: PIPE-250
    
- **제목**: Scenario-to-Strategy Pipeline (V2)
    
- **버전**: 1.0
    
- **목표**: 사용자 입력 시나리오(자연어 가설/예상)를 **구조화(JSON)** 하고, 그 결과를 전략 생성/평가에 **일관된 방식으로 반영**(또는 배제)하여, “시나리오가 어떤 영향을 줬는지”를 **증거(아티팩트/로그)** 로 남긴다.
    
- **범위**: 시나리오 입력→파싱→품질평가→전략 반영→스트레스/검증→저장 규칙.
    
- **비범위**: 시나리오 파서/스코어러 내부 알고리즘(MOD-470/471), 전략 생성기 내부(MOD-410~413)
    

---

## 1) 파이프라인 위치(PIPE-200 내)

PIPE-200 Stage Graph에서 시나리오 파이프는 다음 단계로 매핑된다.

- **S10 Scenario Intake**: MOD-470/471
    
- **S20 Generate Candidates**: MOD-410(+MOD-411)
    
- **S40 DQ Gate**: 데이터 관점 품질(OBS-540)
    
- **S50/S70 Backtest**: 시나리오 반영/미반영 비교 또는 스트레스 테스트(정책에 따라)
    

---

## 2) 입력/출력(Contracts)

### 2.1 입력: `ScenarioInput` (자연어)

- 사용자 텍스트(가설/예상/전제/기간/대상/조건)
    
- 선택 입력(권장):
    
    - 자산군/시장(주식/코인/ETF 등)
        
    - 적용 기간(예: 향후 3개월)
        
    - 확신 수준(낮음/중간/높음)
        

### 2.2 출력: `ScenarioJSON`

- MOD-470 출력(JSON 스키마) + metadata
    
- 예:
    

```json
{
  "scenario_id": "scn_20260206_001",
  "hypothesis": "금리 인하 기대가 커지며 성장주 강세",
  "time_horizon": { "start": "2026-02-01", "end": "2026-05-31" },
  "assets": { "universe_hint": "NASDAQ100", "symbols_hint": [] },
  "drivers": ["rates_down", "risk_on"],
  "constraints_hint": {
    "avoid": ["banks"],
    "prefer": ["growth", "tech"]
  },
  "testable_claims": [
    { "metric": "relative_strength", "direction": "up", "target": "growth_vs_value" }
  ],
  "assumptions": ["연준이 1~2회 인하 신호를 준다"]
}
```

### 2.3 출력: `ScenarioQuality`

- MOD-471 출력:
    
    - 명확성/검증가능성/일관성/과도한 주장/모호성 등 점수 + 판정
        

```json
{ "score": 0.72, "verdict": "ACCEPT|SOFT|IGNORE|REJECT", "reasons": ["MISSING_QUANT_THRESHOLD"] }
```

---

## 3) 품질 판정(Branch Policy)

ScenarioQuality.verdict에 따라 분기한다.

### 3.1 ACCEPT

- 시나리오를 전략 생성/평가에 적극 반영
    
- 반영 방식(아래 4장)에서 “hard/soft” 비율을 높임
    

### 3.2 SOFT

- 시나리오는 “힌트”로만 사용
    
- 전략 공간을 좁히되, 배제하지는 않음(과편향 방지)
    

### 3.3 IGNORE

- 시나리오를 저장만 하고 전략에는 반영하지 않음
    
- 단, 비교/리포트에는 “시나리오 무시됨”을 명시
    

### 3.4 REJECT

- 시나리오 자체가 너무 모호/비검증/자기모순일 때
    
- 정책:
    
    - (권장) iteration은 시나리오 없이 진행 가능하도록 fallback(PIPE-210 Branch)
        
    - 또는 “시나리오 입력 재요청” 플로우(제품 UX)
        

---

## 4) 시나리오 → 전략 반영 방식(Integration Modes)

시나리오가 전략에 들어가는 방식은 “모드”로 고정한다.

### 4.1 Mode A: Candidate Filter (하드 필터)

- 시나리오가 특정 유니버스/섹터/레짐을 요구하는 경우
    
- 예: “리스크온” → `regime_state=RISK_ON`에서만 작동하도록 제약 생성
    
- 장점: 탐색 공간 축소
    
- 단점: 틀린 시나리오면 전체가 망함 → ACCEPT에서만 제한적으로
    

### 4.2 Mode B: Prior/Weighting (소프트 가중)

- Generator 샘플링에서 특정 템플릿/룰/피처 조합의 확률을 올림
    
- 예: 성장주 강세 가설 → RS/모멘텀 계열 템플릿 가중치↑
    

### 4.3 Mode C: Stress Test Overlay (스트레스 테스트)

- “시나리오가 맞는 구간/국면”을 slice로 만들어 성능을 별도로 계산
    
- 예: `drivers=risk_on` → risk_on 마스크 slice 성과/낙폭 측정
    
- 장점: 편향 없이 “시나리오 조건에서만 좋다”를 드러냄
    

### 4.4 Mode D: Constraint Hint (제약 힌트)

- 시나리오가 “피하고 싶은 것/선호”를 말할 때
    
- 예: “은행 피하자” → 섹터 캡/제외 리스트 제약(단, 정책상 soft 적용 가능)
    

> 기본 권장 조합: **SOFT일 때 Mode B + C**, ACCEPT일 때 B + C + (제한적)D

---

## 5) 평가 설계(Scenario-aware Evaluation)

시나리오 반영이 과적합이 되지 않게 평가를 설계한다.

### 5.1 A/B 비교(권장 옵션)

- 동일 전략(또는 동일 패밀리)을
    
    - 시나리오 반영 ON vs OFF로 비교
        
- 산출:
    
    - `delta_score`, `delta_oos`, `delta_dd`
        
- 목적: “시나리오 덕분에 좋아진 건지, 원래 좋은 전략인지” 분리
    

### 5.2 Slice 평가(필수 권장)

- 시나리오 driver 기반 slice를 SCHEMA-340 slices에 기록
    
- 예: `slices.scenario.risk_on.*`
    

---

## 6) 아티팩트/로그 저장(PIPE-220 연동)

### 6.1 저장 위치(권장)

- `artifacts/runs/<run_id>/iterations/<iter>/scenario/`
    
    - `scenario_input.txt`
        
    - `scenario.json` (ScenarioJSON)
        
    - `scenario_quality.json`
        
    - `scenario_integration.json` (아래 6.2)
        

### 6.2 scenario_integration.json (필수 권장)

시나리오가 실제로 “어떻게 적용됐는지”를 남긴다.

```json
{
  "scenario_id": "scn_20260206_001",
  "verdict": "SOFT",
  "modes": ["WEIGHTING", "STRESS_SLICE"],
  "applied_to": {
    "generator": { "template_bias": { "momentum_family": 1.5 } },
    "constraints": { "sector_avoid": ["banks"], "strength": "soft" },
    "evaluation": { "ab_compare": true, "slice_names": ["scenario_risk_on"] }
  }
}
```

---

## 7) 예외/에러 처리

- `SCENARIO_PARSE_FAIL`(MOD-470 실패)
    
- `SCENARIO_SCHEMA_INVALID`(validator 실패)
    
- `SCENARIO_CONFLICTING_CLAIMS`(내부 모순)
    
- `SCENARIO_INTEGRATION_OVERCONSTRAIN`(후보 0으로 만드는 과제약)
    

정책:

- parse/validate 실패 시:
    
    - IGNORE로 다운그레이드 + 저장만
        
- overconstrain 발생 시:
    
    - Mode A/D를 완화하고 Mode B/C 중심으로 재시도(PIPE-210 Branch 기록)
        

---

## 8) 테스트 요구사항

- **TEST-700**: 시나리오 입력→scenario.json/quality/integration 파일 생성
    
- **TEST-701**: verdict=REJECT 시 fallback(시나리오 없이 iteration 진행) 정상
    
- **TEST-702**: overconstrain 발생 시 자동 완화 분기 + 이벤트 로그 기록
    
- **TEST-703**: scenario slice가 SCHEMA-340 slices에 기록됨
    

---

다음은 **PIPE-260 — Paper/Live Execution Pipeline**이야.  
“다음”이라고 하면 PIPE-260만 작성할게.