## Condition AST Specification

- **문서 ID**: SCHEMA-310
    
- **제목**: Condition AST(JSON) Specification
    
- **버전**: 1.0
    
- **목표**: 조건식을 **문자열 파싱 없이** 안전하게 표현/검증/정규화/해시/평가할 수 있는 AST 포맷을 정의한다.
    
- **범위**: `modules.entry/filter/exit`에서 사용되는 모든 조건식. (백테스트/라이브 공통)
    
- **비범위**: 지표 계산 로직(=features 정의)은 SCHEMA-300.
    

---

## 1. 설계 원칙

1. **안전성**: 임의 코드 실행 금지. 허용된 노드/연산자만 사용.
    
2. **정규화 가능**: 동등한 조건식은 같은 canonical form으로 변환 가능해야 함(중복 제거).
    
3. **타입 안정성**: 숫자-숫자, 상태-상태 비교 등 타입 규칙을 명확히.
    
4. **NaN/결측 규율**: 결측이 조건에 들어오면 어떻게 처리하는지 일관되게.
    
5. **설명 가능성**: True/False 뿐 아니라 “어떤 비교가 실패했는지” reason code 생성 가능.
    

---

## 2. AST 문서 구조

AST는 JSON 오브젝트이며, 최소 필드:

- `type` (노드 타입)
    
- 타입별 필수 필드(아래 정의)
    

전략 DSL에서는 `conditions` 블록 아래에 저장하고, 모듈은 `ref`로 참조한다.

---

## 3. 값(Value) 표현 규칙

CMP의 좌/우는 아래 중 하나다.

### 3.1 FeatureRef

- 형식: 문자열
    
- 값: `features`에 정의된 key (예: `"rsi_14"`, `"ema_21"`)
    
- 예외: 시스템 변수(SystemVar)도 문자열로 표현 가능(허용 목록만)
    

### 3.2 Literal

- 숫자: `number`
    
- 문자열(enum): `string`  
    예: `"RISK_ON"`, `"BUY"`, `"Semiconductors"`
    

### 3.3 SystemVar (허용 목록)

다음은 런타임에서 제공되는 “내장 변수”로, `features`에 없어도 참조 가능.

- `regime_state` (string enum)
    
- `regime_score` (number)
    
- `symbol` (string)
    
- `sector` (string) _(가능하면 universe/selection 단계에서 제공)_
    
- `position_qty` (number)
    
- `position_avg_price` (number)
    
- `exposure_weight` (number)
    
- `spread_bps` (number) _(가능하면 execution/liquidity에서 제공)_
    
- `rvol` (number)
    

> 허용 목록 외 문자열을 CMP에 넣으면 validator가 실패 처리.

---

## 4. 노드 타입 정의

## 4.1 CMP (비교 노드) — 필수

두 값을 비교한다.

```json
{ "type": "CMP", "left": "rsi_14", "op": "<=", "right": 30 }
```

### CMP 필드

- `left`: FeatureRef | SystemVar | Literal(권장X)
    
- `op`: `"==" | "!=" | ">" | ">=" | "<" | "<="`
    
- `right`: FeatureRef | SystemVar | Literal
    

### 타입 규칙

- `> >= < <=` 는 **숫자-숫자** 비교만 허용
    
- `== !=` 는 숫자/문자열 모두 허용 (단, 좌/우 타입이 동일해야 함)
    
- FeatureRef는 런타임에서 number로 평가되는 값만 `> >= < <=`에 사용
    

---

## 4.2 AND / OR (논리 결합)

```json
{ "type": "AND", "children": [ ... ] }
{ "type": "OR",  "children": [ ... ] }
```

### 필드

- `children`: ASTNode[] (길이 ≥ 2)
    

### 평가 규칙

- Short-circuit 적용(성능)
    
- 단, **설명 로그가 필요하면** “전체 평가 모드(full_eval=true)”를 지원(옵션)
    
    - full_eval=true일 때는 short-circuit하더라도 모든 child의 결과를 계산해 “어떤 조건이 실패했는지”를 수집 가능
        

---

## 4.3 NOT (부정)

```json
{ "type": "NOT", "child": { ... } }
```

### 필드

- `child`: ASTNode
    

---

## 4.4 IN (집합 포함) — 선택

```json
{ "type": "IN", "left": "sector", "set": ["Semiconductors", "Software"] }
```

### 필드

- `left`: FeatureRef | SystemVar | Literal
    
- `set`: Literal(string|number)[] (길이 ≥ 1)
    

### 타입 규칙

- left 타입과 set 원소 타입이 동일해야 함
    

---

## 4.5 BETWEEN (구간) — 선택(권장)

```json
{ "type": "BETWEEN", "value": "rsi_14", "low": 20, "high": 40, "inclusive": true }
```

### 필드

- `value`: FeatureRef | SystemVar
    
- `low`: number
    
- `high`: number
    
- `inclusive`: boolean (default true)
    

> BETWEEN은 내부적으로 `low <= value <= high`로 변환 가능(정규화 참고).

---

## 4.6 TRUE / FALSE (상수) — 선택

```json
{ "type": "TRUE" }
{ "type": "FALSE" }
```

---

## 5. 결측/NaN 정책 (필수)

AST 평가 시 값이 NaN/None/결측이면 정책에 따라 처리한다.  
정책은 Strategy DSL의 `metadata.nan_policy`를 따른다.

### 5.1 정책 값(표준)

- `DISALLOW_TRADE`: 조건 평가 결과를 False로 처리 + reason code `DATA_MISSING`
    
- `TREAT_AS_FALSE`: False로만 처리(거래 금지만, 코드 없음)
    
- `TREAT_AS_TRUE`: True로 처리(권장하지 않음)
    
- `ERROR`: 평가 자체를 실패(테스트/디버그에만)
    

> 기본 권장: `DISALLOW_TRADE`

### 5.2 적용 범위

- CMP에서 left/right가 NaN이면 정책 적용
    
- AND/OR는 child 결과에 따라 결측이 전파되지 않도록 CMP 단계에서 결정
    

---

## 6. 평가 결과 포맷(Evaluation Output)

AST는 단순 bool만 반환하지 않고, **설명 가능한 결과**를 반환할 수 있어야 한다.

### 6.1 최소 출력

```json
{
  "value": true,
  "failed_clauses": [],
  "passed_clauses": []
}
```

### 6.2 clause 포맷(표준)

CMP 하나를 “clause”로 기록

```json
{
  "node_path": "0.1",                // children 인덱스 경로
  "left": "rsi_14",
  "op": "<=",
  "right": 30,
  "left_value": 29.0,
  "right_value": 30,
  "result": true,
  "reason_code": "RSI_OVERSOLD"
}
```

- `node_path`: 디버그용 (정규화 이후에도 안정적으로 만들려면 canonical id를 별도로 둘 수 있음)
    
- `reason_code`: reason mapping 규칙(섹션 9)
    

---

## 7. 정규화(Canonicalization) 규칙 (중복 제거 핵심)

동일 의미 조건식은 동일 형태로 변환하여 해시를 같게 만든다.

### 7.1 AND/OR 평탄화(flatten)

- AND 안에 AND가 있으면 children을 한 레벨로 올림
    
- OR도 동일
    

### 7.2 children 정렬(sort)

- AND/OR의 children은 **canonical key**로 정렬  
    canonical key는 아래 우선순위를 추천:
    
    1. node type
        
    2. CMP면 `(left, op, right)` 문자열화
        
    3. IN이면 `(left, sorted(set))`
        
    4. BETWEEN이면 `(value, low, high, inclusive)`
        
    5. NOT이면 child의 key
        

### 7.3 중복 제거(dedup)

- AND/OR 내 동일 child가 2개 이상이면 하나만 유지
    

### 7.4 상수 축약(constant folding) — 선택

- AND에 FALSE가 있으면 전체 FALSE
    
- OR에 TRUE가 있으면 전체 TRUE
    
- AND에 TRUE만 있으면 TRUE
    
- OR에 FALSE만 있으면 FALSE
    

### 7.5 BETWEEN 정규화

- BETWEEN은 내부적으로 CMP 2개로 변환 가능(선택)
    
    - 다만 원본 보존이 필요하면 `normalized_from` 메타 필드로 추적
        

---

## 8. 해시(Strategy ID / Condition Hash) 규칙

### 8.1 목적

- 중복 전략 제거
    
- 동일 조건식 여부 판단
    

### 8.2 방법(권장)

1. AST를 정규화(canonicalize)
    
2. JSON을 **키 정렬 + 공백 제거** 형태로 직렬화
    
3. SHA-256 해시
    
4. 해시 앞 12~16자를 `condition_id`로 사용 가능
    

> 구현은 MOD-412(Strategy Normalizer & Hash)에 명시.

---

## 9. Reason Code 매핑(설명 가능성 표준)

AST 평가에서 reason code를 생성하려면 “비교 패턴 → 코드” 매핑이 필요하다.

### 9.1 기본 매핑 규칙(권장)

- CMP의 `left`가 특정 feature이면 자동 코드 부여 가능  
    예:
    
    - `left startswith "rsi_"` and `op "<="` → `RSI_OVERSOLD`
        
    - `left startswith "rsi_"` and `op ">="` → `RSI_OVERBOUGHT`
        
    - `left startswith "stoch_k_"` and `op ">="` → `STOCH_HIGH`
        
    - `left startswith "stoch_k_"` and `op "<="` → `STOCH_LOW`
        
    - `left startswith "adx_"` and `op ">="` → `ADX_OK`
        
    - `left == "regime_state" and op "==" and right "RISK_ON"` → `FILTER_OK` (또는 `REGIME_RISK_ON`)
        

### 9.2 명시적 매핑(선택, 더 정확)

CMP 노드에 `reason_code`를 직접 박을 수 있다(선택 필드).

```json
{ "type":"CMP", "left":"adx_14", "op":">=", "right":20, "reason_code":"ADX_OK" }
```

- 장점: 지표명이 바뀌어도 코드 안정
    
- 단점: DSL이 장황해짐 → Generator가 자동 부여하도록 권장
    

---

## 10. Validator 요구사항(필수 체크리스트)

### 10.1 구조 검증

- 모든 노드에 `type` 존재
    
- AND/OR는 children 길이 ≥ 2
    
- NOT은 child 1개
    
- CMP는 left/op/right 존재, op는 허용된 값
    

### 10.2 참조 검증

- CMP의 left/right가 문자열일 때:
    
    - `features`에 존재하거나 SystemVar 허용 목록에 있어야 함
        
- IN의 set은 빈 배열 금지
    

### 10.3 타입 검증

- `> >= < <=`는 number 비교만
    
- `== !=`는 좌/우 타입 일치
    

### 10.4 복잡도 제한(초기 기본값)

- max_depth ≤ 4
    
- max_cmp_count ≤ 8
    
- max_children_per_node ≤ 8
    

### 10.5 NaN 정책 강제

- `metadata.nan_policy`가 없으면 기본 `DISALLOW_TRADE`로 간주(또는 validator가 경고)
    

---

## 11. 예시 모음

### 11.1 요청 예시: Stoch ≥ 80 AND RSI ≤ 30

```json
{
  "type": "AND",
  "children": [
    { "type": "CMP", "left": "stoch_k_14_3_3", "op": ">=", "right": 80, "reason_code": "STOCH_HIGH" },
    { "type": "CMP", "left": "rsi_14", "op": "<=", "right": 30, "reason_code": "RSI_OVERSOLD" }
  ]
}
```

### 11.2 EMA 정배열 + ADX + DI

```json
{
  "type": "AND",
  "children": [
    { "type":"CMP", "left":"adx_14", "op":">=", "right":20, "reason_code":"ADX_OK" },
    { "type":"CMP", "left":"di_plus_14", "op":">", "right":"di_minus_14", "reason_code":"DI_BULL" },
    {
      "type":"AND",
      "children":[
        { "type":"CMP", "left":"ema_8", "op":">", "right":"ema_21", "reason_code":"EMA_STACK_OK" },
        { "type":"CMP", "left":"ema_21", "op":">", "right":"ema_55", "reason_code":"EMA_STACK_OK" }
      ]
    }
  ]
}
```

### 11.3 레짐 필터: RISK_ON에서만 거래

```json
{ "type":"CMP", "left":"regime_state", "op":"==", "right":"RISK_ON", "reason_code":"FILTER_OK" }
```

---

## 12. 다음 문서 연결(Traceability)

- AST를 실제로 평가하는 런타임(엔진)은 `MOD-420 Backtester` 및 `MOD-410 Generator`에서 구현
    
- AST 정규화/해시는 `MOD-412 Strategy Normalizer & Hash`에 상세화
    
- reason code 사전은 `DOC-003 / configs/reason_codes.yaml`로 통일
    
- NaN/룩어헤드 정책의 자동 검사 결과는 `OBS-540 Data Quality Checks`로 기록
    

---
