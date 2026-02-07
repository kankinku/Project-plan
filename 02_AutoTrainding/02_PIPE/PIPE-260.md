## Paper/Live Execution Pipeline

- **문서 ID**: PIPE-260
    
- **제목**: Paper/Live Execution Pipeline
    
- **버전**: 1.0
    
- **목표**: 검증된 전략(T1~)을 **paper → live** 로 안전하게 운영하기 위한 실행 파이프라인을 정의한다. 주문 멱등성, 리스크 가드, 관측 로그(증거), 장애 복구까지 포함해 “돌아가긴 하는데 왜 망했는지 모르는” 상태를 방지한다.
    
- **범위**: 신호→주문 의도→OMS→브로커→체결→포트폴리오→리스크 가드→관측→리플레이 입력까지의 운영 플로우.
    
- **비범위**: 개별 전략 로직(진입/청산 규칙), 백테스트 엔진 상세
    

---

## 1) 파이프라인 위치(전체 구조)

- 백테스트 파이프(PIPE-200) 결과 중 **T3(Paper Ready)** 이상만 PIPE-260에 투입(PIPE-240 연동 권장)
    
- 실행 모듈 맵:
    
    - Broker Adapter: MOD-480
        
    - OMS: MOD-481
        
    - Risk Guard: MOD-482
        
    - Replay Engine: MOD-483
        
    - Execution/Portfolio 모델(페이퍼 시뮬용): MOD-424/425(옵션)
        

---

## 2) 실행 단계(Stage Flow)

### L00 Preflight (운영 시작 전)

- 입력: `strategy_id`, policy_versions, mode(paper/live), account_ref
    
- 체크:
    
    - SYS-130 가드(권한/키/마스킹/멱등성 설정)
        
    - DQ 상태(최근 데이터 이상 여부)
        
    - Risk Guard 파라미터 로드
        
- 출력: `session_id`, 운영 메타 기록(OBS-530 확장 또는 별도 ops meta)
    

### L10 Market Data Tick / Bar Close

- 입력: 실시간 바/틱(또는 일정 주기 스냅샷)
    
- 처리: 타임존/캘린더 정합성 확인
    
- 출력: `market_snapshot`
    

### L20 Strategy Decision (Signal)

- 입력: market_snapshot + 현재 포지션 상태
    
- 출력: `decision_event`(OBS-500 형식 권장)
    
    - “왜 샀는지/왜 안 샀는지” reason_codes 포함
        

### L30 Order Intent Build

- 입력: decision_event, sizing policy, constraints(포트폴리오/리스크)
    
- 출력: `order_intent`
    
- 사전 차단:
    
    - 포트폴리오 제약 위반(예: max positions, sector cap)
        
    - 스프레드/ADV 등 실행 제약(가능하면 pre-check)
        

### L40 OMS Submit (Idempotent)

- 입력: order_intent
    
- 처리:
    
    - `client_order_id` 생성(멱등성 키)
        
    - 재시도 정책 적용(PIPE-210 준용)
        
- 출력: `order_ack` + OMS 상태 업데이트
    

### L50 Broker Execution / Fill

- 입력: OMS 주문
    
- 출력: fills / rejects / partials
    
- 기록: OBS-510(주문/체결/거절/지연/부분체결)
    

### L60 Portfolio Update

- 입력: fills
    
- 처리: 현금/포지션/실현손익 업데이트
    
- 출력: `portfolio_snapshot`
    
- 기록: (권장) 포트폴리오 스냅샷 로그(또는 OBS-500 확장)
    

### L70 Risk Guard Check (Post-trade + Continuous)

- 입력: portfolio_snapshot + execution_quality + DQ health
    
- 처리:
    
    - 트립 조건 검사(일 손실/최대 DD/슬리피지 폭증/거절 연속 등)
        
- 출력:
    
    - `OK` 또는 `TRIP`
        
- TRIP 시:
    
    - 신규 주문 중단 + (정책) 오더 취소/청산
        

### L80 End-of-Period Summary

- 입력: 하루/세션 종료 시점
    
- 출력:
    
    - 일간 요약(거래 수, 비용, 슬리피지, DD)
        
    - OBS-520(트레이드 요약) 업데이트/확정
        

---

## 3) 모드별 차이(Paper vs Live)

### 3.1 Paper 모드

- 브로커가 “실체결”을 주지 않는 환경이면:
    
    - MOD-424 Execution Model로 체결을 시뮬레이션(옵션)
        
- 목적:
    
    - 주문/체결 파이프라인이 운영적으로 안정적인지 검증
        
    - 로그가 충분히 쌓이는지 확인(리플레이 가능성 확보)
        

### 3.2 Live 모드

- 실체결 기반(브로커 응답이 사실의 근원)
    
- 필수:
    
    - SYS-130 안전장치 전부
        
    - 멱등성(client_order_id) 강제
        
    - Risk Guard always-on
        

---

## 4) 멱등성/재시도 정책(OMS 핵심)

### 4.1 client_order_id 생성 규칙(권장)

`client_order_id = hash(run_or_session_id, strategy_id, symbol, side, decision_time, intent_hash)`

### 4.2 재시도 분류

- `TRANSIENT`(timeout/5xx 등):
    
    - 동일 client_order_id로 재시도(중복 주문 방지)
        
- `PERMANENT`(잔고부족/거절 코드):
    
    - 재시도 금지, 실패 근거 기록(OBS-510 reject)
        

### 4.3 중복 방지 원칙

- “재전송”은 주문 새로 만들기 아님
    
- OMS는 **이미 제출된 주문의 상태 조회/동기화**로 처리
    

---

## 5) 관측(Observability) 필수 로그

### 5.1 최소(권장 필수)

- OBS-500: decision(바/틱 또는 바 종료 단위)
    
- OBS-510: order/fill (브로커 사실)
    
- OBS-520: trade summary(트레이드 확정)
    
- (운영 메타) 세션 메타(키 마스킹, 정책 버전, 모드, 계좌 식별자 마스킹)
    

### 5.2 실행 품질 지표(실시간 모니터)

- reject_rate, partial_fill_rate
    
- slippage bps p50/p95
    
- latency ms p95
    
- ADV participation, spread bps(가능 시)
    

---

## 6) 장애/복구(Recovery)

### 6.1 프로세스 재시작 시

- 브로커/OMS에서:
    
    - 열린 주문(open orders) 재조회
        
    - 현재 포지션 재조회
        
- 내부 상태 복구:
    
    - 마지막 portfolio_snapshot 로드
        
    - 미확정 트레이드 상태 복원
        

### 6.2 로그 기반 복구

- OBS-510이 사실의 근원:
    
    - fills를 기준으로 포트폴리오를 재구성(log-driven replay)
        
- 복구 완료 후에만 신규 주문 허용
    

---

## 7) 안전장치 트립 시 동작(Stop Plan)

- 신규 주문 금지(즉시)
    
- (정책) 모든 오픈 오더 취소
    
- (정책) 포지션 축소/청산 여부 결정(보수적으로는 축소)
    
- 트립 이벤트 기록:
    
    - 원인 코드 + 임계값 + 당시 포트폴리오 상태 + 실행 품질
        

---

## 8) 아티팩트/저장(PIPE-220 연동)

권장 경로:

- `artifacts/runs/<run_id>/live/` 또는 `artifacts/ops/sessions/<session_id>/`
    
    - `ops_metadata.json`
        
    - `obs_decisions.jsonl` (OBS-500)
        
    - `obs_orders_fills.jsonl` (OBS-510)
        
    - `obs_trades.jsonl` (OBS-520)
        
    - `risk_events.jsonl` (trip 기록)
        

---

## 9) 예외/에러 코드(권장)

- `LIVE_GUARD_NOT_ENABLED` (Risk Guard 비활성 시 실행 차단)
    
- `OMS_IDEMPOTENCY_MISSING`
    
- `BROKER_REJECTED`
    
- `BROKER_TIMEOUT_TRANSIENT`
    
- `POSITION_SYNC_FAIL`
    
- `RISK_GUARD_TRIPPED`
    

---

## 10) 테스트 요구사항

- **TEST-710**: paper 모드 E2E(결정→주문→체결 시뮬→포트폴리오→로그)
    
- **TEST-711**: live 모드에서 Risk Guard 비활성 → 실행 차단
    
- **TEST-712**: 동일 client_order_id 재시도 → 중복 주문 0
    
- **TEST-713**: 네트워크 장애 후 재시작 → open orders/positions 동기화 + 정상 복구
    
- **TEST-714**: Risk Guard 트립 → 신규 주문 0 + 이벤트 로그 생성
    

---
