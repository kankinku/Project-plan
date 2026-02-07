## Parallelism & Batch Scheduling

- **문서 ID**: PIPE-230
    
- **제목**: Parallelism & Batch Scheduling
    
- **버전**: 1.0
    
- **목표**: 대량 전략 후보를 처리할 때(특히 S50 Backtest A, S70 Backtest B) **병렬 처리·배치 스케줄링·리소스 제한**을 표준화해서, 속도는 확보하면서도 **재현성/관측/안전장치**를 깨지 않도록 한다.
    
- **범위**: 작업 큐/배치 단위, 병렬 모델, 리소스 예산(메모리/CPU), 결정성(seed), 실패 격리, 체크포인트/리줌.
    
- **비범위**: 분산 인프라 선택(쿠버네티스/레이 등)은 구현 단계에서 결정(여기서는 “정책/계약”만)
    

---

## 1) 병렬화 대상과 원칙

### 1.1 병렬화 대상

- **S50 Backtest A**: 전략 단위 병렬(가장 큰 처리량 필요)
    
- **S70 Backtest B**: 전략 단위 병렬(단, 로그/상태가 무거우므로 동시성 제한)
    
- (옵션) Feature 계산: 피처 셋 단위 병렬(캐시 키 중심)
    

### 1.2 원칙

- 병렬화는 **전략 단위 격리**가 기본(전략 간 상태 공유 금지)
    
- 동일 입력이면 결과가 바뀌지 않도록 **결정성(seed 고정)** 유지(MOD-401)
    
- 실패는 “전체”가 아니라 **전략 단위로 격리**(PARTIAL 허용)
    

---

## 2) 작업 단위(Work Unit) 정의

### 2.1 WorkUnit 스키마(권장)

```json
{
  "run_id": "uuid",
  "iteration_id": 3,
  "stage_id": "S50",
  "strategy_id": "hash",
  "engine": "A|B",
  "data_ref": { "data_version": "data_v2026_02_01", "integrity_hash": "..." },
  "calc_contract_ref": "configs/calc_contract.json",
  "constraint_set_id": "cset_12",
  "policy_versions": { "score": "score_v1", "exec": "exec_v1", "cost": "cost_v1", "pf": "pf_v1" },
  "seed": 123456789
}
```

### 2.2 Batch(배치) 정의

- 배치는 WorkUnit의 묶음
    
- 배치 크기 기본값(권장):
    
    - Engine A: `batch_size = 50~200`
        
    - Engine B: `batch_size = 1~10` (로그/상태 무거움)
        

---

## 3) 스케줄링 정책(Scheduling Policy)

### 3.1 우선순위(Priority)

- 기본 우선순위:
    
    1. **Engine B shortlist**
        
    2. **Engine A 상위 점수 후보**
        
    3. 나머지 후보
        
- 목적: 빠르게 “가치 있는 후보”를 검증하고, 불필요한 대량 계산을 줄임
    

### 3.2 동시성(Concurrency) 제한

- run 단위로 상한을 고정(OBS-530 기록 권장)
    
    - `max_workers_A`: CPU 코어 기반(예: 코어-1)
        
    - `max_workers_B`: 보수적으로(예: 1~4)
        
- 메모리 상한 기반 동적 제한:
    
    - `if free_mem < threshold: reduce_workers()`
        

### 3.3 리소스 예산(Resource Budget)

- 각 worker는 “예상 메모리 비용”을 선언:
    
    - Engine A: feature cache 공유 가능(프로세스 내)
        
    - Engine B: per-strategy state가 커서 공유 최소화
        
- 예산 초과 시:
    
    - batch 크기 축소 또는 workers 축소(분기) + 이벤트 로그
        

---

## 4) 결정성(Determinism) & Seed 정책

- WorkUnit의 seed는 **파생 규칙으로 고정**:
    
    - `seed = hash(run_id, iteration_id, stage_id, strategy_id, policy_versions, data_integrity_hash)`
        
- 병렬 실행 순서가 달라져도 결과가 같아야 함:
    
    - 랜덤 요소(지연/부분체결)는 seed 기반 샘플링
        
- seed/파생 규칙은 OBS-530에 기록(또는 pipeline 이벤트로 기록)
    

---

## 5) 체크포인트 & 리줌(Checkpoint/Resume)

### 5.1 저장 단위

- 최소 단위: **전략 단위 산출물**
    
    - metrics 파일(전략별 SCHEMA-340)
        
    - (검증) logs append + index 업데이트
        
- 배치 완료 마커:
    
    - `pipeline/.S50_BATCH_<k>_DONE` 같은 마커 생성(옵션)
        

### 5.2 리줌 규칙

- strategy_id 기준으로 “이미 산출된 결과”가 있으면:
    
    - 입력 해시가 같을 때만 스킵(PIPE-220 inputs_hash)
        
- 입력이 바뀌면:
    
    - 스킵 금지 → 재실행 + 이벤트 기록
        

---

## 6) 실패 격리(Failure Isolation)

### 6.1 전략 단위 실패 처리

- WorkUnit 실패는 해당 전략만 `FAILED`로 표시하고 나머지 계속
    
- 실패 원인 분류:
    
    - `TRANSIENT`(재시도 가능)
        
    - `PERMANENT`(전략 폐기)
        
    - `CRITICAL`(run 중단 트리거 가능)
        

### 6.2 배치 실패 처리

- 배치 내 실패율이 임계 초과(예: 50%)하면:
    
    - “공통 원인” 가능성이 큼(데이터/설정)
        
    - 배치를 중단하고 DQ/설정 점검 브랜치로 전환(PIPE-210)
        

---

## 7) 로깅(Observability) & 모니터링

- pipeline 이벤트(권장): `pipeline_events.jsonl`
    
    - 배치 시작/완료, worker 수 변경, 리소스 압박, 재시도
        
- 운영 지표(권장):
    
    - throughput(전략/분), 평균/95p 소요시간
        
    - 실패율(TRANSIENT/PERMANENT)
        
    - OOM/timeout 카운트
        
    - 캐시 적중률(feature cache hit ratio, 가능 시)
        

---

## 8) 구현 선택지(비결정 영역)

이 문서는 “정책”만 고정하고 구현은 선택 가능:

- 단일 머신:
    
    - multiprocessing / joblib / concurrent futures
        
- 분산:
    
    - Ray / Celery / Batch 시스템
        
- 단, WorkUnit 계약/seed/아티팩트 규칙은 동일하게 유지해야 함
    

---

## 9) 예외/에러 코드(권장)

- `SCHED_OOM_PRESSURE` (worker 축소 트리거)
    
- `SCHED_TIMEOUT` (WorkUnit 타임아웃)
    
- `SCHED_RETRY_BUDGET_EXCEEDED`
    
- `SCHED_INPUT_HASH_MISMATCH` (리줌 스킵 불가)
    
- `SCHED_BATCH_FAIL_RATE_HIGH`
    

---

## 10) 테스트 요구사항

- **TEST-680**: 병렬 실행 순서가 달라도 결과 동일(결정성)
    
- **TEST-681**: OOM 압박 시 worker 수 자동 축소 + 이벤트 기록
    
- **TEST-682**: 중간 크래시 후 resume 시 중복 산출물 없이 이어서 완료
    
- **TEST-683**: 전략 단위 실패가 전체 파이프라인을 멈추지 않음(PARTIAL 유지)
    
