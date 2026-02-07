## Stop / Retry / Branch Policy

- **문서 ID**: PIPE-210
    
- **제목**: Stop, Retry, and Branch Policies
    
- **버전**: 1.0
    
- **목표**: PIPE-200의 Stage Graph 실행 중 발생하는 실패/부분실패/품질 이슈에 대해 **중단(Stop)**, **재시도(Retry)**, **분기(Branch)** 기준을 고정해 “무한 반복”이더라도 **안전하고 재현 가능하게** 운영한다.
    
- **범위**: 단계 상태 정의, 에러 분류, 재시도 예산/백오프, 중단 조건, 분기(대체 경로), 멱등성(재실행 안전) 규칙.
    
- **비범위**: 각 모듈 내부 알고리즘(각 MOD 문서), 보안/권한(SYS-130)
    

---

## 1) 입력 / 출력

### 1.1 입력

- PIPE-200 Stage 실행 컨텍스트:
    
    - `run_id`, `iteration_id`, `stage_id`
        
    - `mode(backtest/paper/live)`
        
    - `policy_versions(score/cost/exec/pf)`
        
    - `data_ref`, `calc_contract`
        
    - `constraint_set_id`
        
- 에러 이벤트:
    
    - `error_code`, `error_class`, `message`, `details`
        

### 1.2 출력

- Stage 실행 결과:
    
    - `stage_status`
        
    - `retry_decision`(retry/stop/branch)
        
    - `next_stage_id`
        
- 파이프라인 이벤트 로그(권장):
    
    - `pipeline_events.jsonl` (stage transition, stop reason, retry attempts)
        

---

## 2) 상태(State) 정의

### 2.1 StageStatus enum

- `PENDING`: 실행 대기
    
- `RUNNING`: 실행 중
    
- `SUCCESS`: 성공
    
- `FAILED`: 실패(복구 불가)
    
- `PARTIAL`: 부분 성공(일부 전략/심볼/기간만 성공)
    
- `SKIPPED`: 정책상 스킵(예: scenario 단계 비활성)
    
- `RETRYING`: 재시도 중
    
- `STOPPED`: 중단(런/이터레이션 종료)
    

### 2.2 ErrorClass enum

- `TRANSIENT`: 일시 오류(네트워크, 파일락, 일시적 OOM 등) → 재시도 후보
    
- `PERMANENT`: 입력/정책/스키마 문제 → 재시도 의미 없음
    
- `CRITICAL`: 신뢰성/안전성 붕괴(룩어헤드, 서바이버십, live risk guard 등) → 즉시 중단
    

---

## 3) 로그(Logging) 규칙

### 3.1 필수(권장) 파이프라인 이벤트 로그

`pipeline_events.jsonl`에 아래 이벤트를 기록:

- `STAGE_STARTED`
    
- `STAGE_COMPLETED`(SUCCESS/PARTIAL)
    
- `STAGE_FAILED`
    
- `STAGE_RETRY_SCHEDULED`
    
- `STAGE_STOPPED`
    
- `BRANCH_TAKEN`
    

각 레코드 최소 필드:

- `run_id`, `iteration_id`, `stage_id`
    
- `status_before`, `status_after`
    
- `attempt`, `max_attempts`
    
- `error_code`(있으면), `stop_reason`(있으면)
    
- `ts`
    

### 3.2 OBS 연동

- run/iteration 레벨 요약은 **OBS-530**의 `pipeline_summary`에 집계 가능(옵션)
    
- DQ 관련 stop은 **OBS-540**의 결과를 근거로 링크
    

---

## 4) 재시도(Retry) 정책

### 4.1 재시도 예산(Budget) 기본값

- Stage별 기본 최대 시도 횟수(권장):
    
    - S40 DQ Gate: `1` (원칙적으로 재시도 의미 적음)
        
    - S50 Backtest A: `2` (엔진 일시 오류 대비)
        
    - S70 Backtest B: `2` (엔진/리소스 오류 대비)
        
    - S95 Persist: `3` (저장 실패는 재시도 가치 높음)
        
    - 그 외: `1~2`
        
- Run 전체 재시도 상한(권장):
    
    - `max_total_retries_per_iteration = 5`
        

### 4.2 백오프(Backoff)

- `TRANSIENT`에만 적용
    
- 권장: 지수 백오프 + 지터
    
    - 1회: 1~2s
        
    - 2회: 2~5s
        
    - 3회: 5~15s
        
- 백오프 정책은 run metadata(OBS-530)에 기록 권장
    

### 4.3 재시도 조건

- 재시도 **허용**:
    
    - `DATA_LOAD_FAIL`(일시적 IO)
        
    - `TIMEOUT`(리소스 부족, 단 제한적인 1회)
        
    - `OUT_OF_MEMORY`(설정 다운스케일 분기와 함께)
        
    - `DB_LOCKED`(persist 단계)
        
- 재시도 **금지**(PERMANENT/CRITICAL):
    
    - `LOOKAHEAD_POLICY_VIOLATION`
        
    - `SURVIVORSHIP_CHECK_FAIL`
        
    - `SCHEMA_MISMATCH`(입력 자체 오류)
        
    - `INVALID_CONFIG`
        
    - live에서 `RISK_GUARD_TRIPPED`
        

---

## 5) 중단(Stop) 정책

### 5.1 즉시 중단(CRITICAL Stop)

아래는 **즉시 STOPPED**:

- DQ CRITICAL:
    
    - 룩어헤드, 서바이버십, 캘린더/타임존 붕괴 등(OBS-540 기반)
        
- live/paper에서 안전장치 트립:
    
    - Risk Guard(MOD-482) 트립, idempotency 위반 의심 등
        
- 포트폴리오 상태 붕괴:
    
    - `PORTFOLIO_STATE_CORRUPT`(정합성 깨짐)
        
- 스키마/버전 불일치로 재현성 붕괴:
    
    - `SCHEMA_MISMATCH` + 정책이 “strict”
        

### 5.2 Iteration 중단(soft stop)

- shortlist 0개가 **연속 N회**(예: 3회)
    
- 후보 생성량이 임계 이하(예: deduped < 10) 연속
    
- 개선 없음 plateau 조건(PIPE-200 종료 트리거 참고)
    
- Engine B 실패율이 임계 초과(예: shortlist의 70% 이상 FAIL)
    

### 5.3 Run 중단 시 “최소 아티팩트 보장”

STOPPED라도 아래는 저장되어야 함:

- OBS-530
    
- OBS-540(가능하면)
    
- iteration_summary(실패 사유 포함)
    

---

## 6) 분기(Branch) 정책

### 6.1 S10 Scenario Intake 분기

- scenario 품질 낮음(MOD-471):
    
    - 정책 `reject`: S10에서 STOP(또는 scenario 없이 진행)
        
    - 정책 `ignore`: scenario를 **미적용**하고 S20으로 진행
        
    - 정책 `soft`: scenario를 “제약/가중치 힌트”로만 약하게 반영
        

### 6.2 S20/S30 (후보 0개 / deduped 0개)

- Branch A: 샘플러 범위 확장(MOD-411 range widen)
    
- Branch B: 제약 완화(최근에 추가된 constraint를 임시 비활성)
    
- Branch C: 템플릿 패밀리 변경(다른 family로 전환)
    
- Branch D: “간단한 baseline 템플릿” 1개 강제 투입(탐색 리셋)
    

### 6.3 S60 shortlist 0개

- Branch A: scoring 게이트 완화(탐색 모드에서만)
    
- Branch B: score weight 조정(정책 버전 변경은 금지, **동일 run에서는 고정** 권장)
    
- Branch C: 후보 수 늘리기(Generate 재수행)
    
- Branch D: DQ/Execution 페널티가 지나치게 강한지 진단 후 제한적으로 완화(단, 기록 필수)
    

### 6.4 S70 (Engine B) 대량 실패

- 원인별 분기:
    
    - Execution Failure 다수 → exec model/constraints 점검(스프레드/ADV cap) + 후보 축소
        
    - Data Failure → 해당 심볼/기간 제외(부분 성공 정책) + DQ 강화
        
    - Risk Model Failure → stop/trail 파라미터 탐색 범위 조정
        

### 6.5 S95 Persist 실패

- Branch A: 로컬 파일 fallback 저장(JSONL) 후 DB 재시도
    
- Branch B: write-ahead log로 큐잉 후 다음 iteration 시작 전 재처리
    

---

## 7) 멱등성(Idempotency) & 재실행 안전 규칙

### 7.1 Stage 멱등성 키

- 각 stage 실행은 `stage_run_id = hash(run_id, iteration_id, stage_id, attempt, inputs_hash)`로 식별(권장)
    
- 동일 `stage_run_id`로 재실행 시:
    
    - 동일 결과 파일이 있으면 “재사용(resume)” 가능
        

### 7.2 원자적(Atomic) 아티팩트 저장

- 파일 저장은 `tmp → rename` 방식으로 원자화
    
- stage 완료 마커 파일(예: `.stage_done`)를 남겨 resume 시점 판단(권장)
    

### 7.3 재시도 시 입력 고정

- retry는 “동일 입력”으로 실행하는 게 원칙
    
- 입력을 바꾸는 것은 retry가 아니라 **branch**로 분류하고 이벤트 로그에 남김
    

---

## 8) 예외(Exceptions) 표준

- `PIPE_POLICY_VIOLATION`: 정책상 허용되지 않는 분기/재시도 시도
    
- `PIPE_RETRY_BUDGET_EXCEEDED`: stage 또는 iteration retry 예산 초과
    
- `PIPE_STOPPED_CRITICAL`: CRITICAL stop 트리거
    
- `PIPE_BRANCH_LOOP_DETECTED`: 동일 분기 반복으로 무한루프 감지
    

---

## 9) 테스트(Test) 요구사항

- **TEST-620**: TRANSIENT 오류 주입 → 재시도 후 성공(백오프/attempt 기록 확인)
    
- **TEST-621**: PERMANENT 오류 주입 → 재시도 없이 FAILED
    
- **TEST-622**: DQ CRITICAL(LOOKAHEAD) → 즉시 STOPPED + OBS-540 근거 기록
    
- **TEST-623**: shortlist 0개 → Branch 실행 + branch event 로그 기록
    
- **TEST-624**: persist 실패 → fallback 저장 후 재시도 성공
    

---
