## Iteration Loop & Stage Graph

- **문서 ID**: PIPE-200
    
- **제목**: Iteration Loop & Stage Graph
    
- **버전**: 1.0
    
- **목표**: Orchestrator(MOD-400)가 수행하는 “한 번의 Iteration(생성→평가→분석→제약→축적)”을 **단계 그래프(Stage Graph)** 로 정의하고, 각 단계의 입력/출력/중단/재시도/아티팩트 저장 규칙을 고정한다.
    
- **범위**: Iteration 파이프라인 정의, 상태 전이, 스톱/재시도 정책, 단계별 산출물.
    
- **비범위**: 각 단계 내부 알고리즘 상세(MOD 문서에서 정의)
    

---

## 1) Stage Graph 개요

### 1.1 단계 목록(기본)

0. **S00 InitRun**
    
1. **S10 Scenario Intake (옵션)**
    
2. **S20 Generate Candidates**
    
3. **S30 Normalize & Dedup**
    
4. **S40 Data Quality Gate**
    
5. **S50 Backtest A (Exploration)**
    
6. **S60 Score & Shortlist**
    
7. **S70 Backtest B (Verification)**
    
8. **S80 Analyze Failures & Build Evidence**
    
9. **S90 Write/Apply Constraints**
    
10. **S95 Persist Knowledge**
    
11. **S99 EndIteration**
    

> 기본 흐름: S20→S30→S40→S50→S60→S70→S80→S90→S95

---

## 2) Stage별 입력/출력 계약

### S00 InitRun

- **입력**: run_config (mode, seed, data_ref, policy_versions)
    
- **출력**: `run_id`, `iteration_id=1`, OBS-530 생성
    
- **중단 조건**: run_config invalid
    
- **아티팩트**: `artifacts/runs/<run_id>/run_metadata.json` (OBS-530)
    

---

### S10 Scenario Intake (옵션, V2)

- **입력**: 사용자 시나리오 NL
    
- **출력**: scenario_json (MOD-470), scenario_quality (MOD-471)
    
- **중단 조건**: quality가 임계 이하 + 정책이 “reject”인 경우
    
- **아티팩트**: `scenario.json`, `scenario_quality.json`
    

---

### S20 Generate Candidates

- **입력**: base templates/atoms (MOD-413), constraints(optional), scenario_json(optional)
    
- **출력**: candidates.jsonl (SCHEMA-300 전략 스펙 후보들)
    
- **중단 조건**: 후보 0개
    
- **아티팩트**: `candidates.jsonl`
    

---

### S30 Normalize & Dedup

- **입력**: candidates.jsonl
    
- **출력**: deduped.jsonl (+ strategy_id 부여, 중복 제거)
    
- **중단 조건**: deduped 0개
    
- **아티팩트**: `deduped.jsonl`, `normalize_stats.json`
    

---

### S40 Data Quality Gate

- **입력**: data_ref, universe_version, calc_contract, (옵션) required_features 통합 목록
    
- **출력**: DQ 결과(OBS-540) + gate decision(pass/fail)
    
- **중단 조건(기본)**:
    
    - LOOKAHEAD / SURVIVORSHIP 등 CRITICAL 발견 시 iteration 중단
        
- **아티팩트**: `dq_checks.jsonl` (OBS-540)
    

---

### S50 Backtest A (Exploration)

- **입력**: deduped 전략들, Engine A(MOD-421) + cost/execution/portfolio 모델(경량 설정)
    
- **출력**: 전략별 metrics (SCHEMA-340), trade summary(OBS-520 최소)
    
- **중단 조건**: Engine A 전체 실패(부분 실패는 허용)
    
- **아티팩트**:
    
    - `metrics/<strategy_id>.json` (SCHEMA-340)
        
    - `logs/trades.jsonl` (OBS-520, 정책에 따라 분리 저장 가능)
        

---

### S60 Score & Shortlist

- **입력**: SCHEMA-340 metrics + scoring policy (MOD-431)
    
- **출력**: shortlist.jsonl (전략 id + score + gate 결과)
    
- **중단 조건**: shortlist 0개(정책에 따라 iteration 종료 또는 재생성 루프)
    
- **아티팩트**: `shortlist.jsonl`, `scores.jsonl`
    

---

### S70 Backtest B (Verification)

- **입력**: shortlist 전략, Engine B(MOD-422) + MOD-423/424/425 풀 설정
    
- **출력**: 검증 metrics (SCHEMA-340), OBS-500/510/520 풀 로그
    
- **중단 조건**:
    
    - DQ CRITICAL 재발견
        
    - 포트폴리오 상태 붕괴/엔진 크래시(복구 불가)
        
- **아티팩트**:
    
    - `metrics_verified/<strategy_id>.json`
        
    - `logs/decisions.jsonl` (OBS-500)
        
    - `logs/orders_fills.jsonl` (OBS-510)
        
    - `logs/trades_verified.jsonl` (OBS-520)
        

---

### S80 Analyze Failures & Build Evidence

- **입력**: 검증 결과(metrics + OBS 로그)
    
- **출력**: failure tags(MOD-440), evidence pack(MOD-442)
    
- **중단 조건**: 없음(실패해도 “분석 실패”로 기록)
    
- **아티팩트**: `analysis/failures.jsonl`, `analysis/evidence/…`
    

---

### S90 Write/Apply Constraints

- **입력**: failures + evidence + 현재 constraint_set
    
- **출력**: 새 constraint_set(또는 업데이트), 적용 결과 로그
    
- **중단 조건**: 제약이 “전부 막아버리는” 경우(탐색 공간 0) → 안전장치 발동(일부 제약 비활성/만료)
    
- **아티팩트**: `constraints/constraint_set_<id>.json`, `constraints/events.jsonl`
    

---

### S95 Persist Knowledge

- **입력**: 최종 shortlist/verified 전략, metrics, failures, constraints
    
- **출력**: KB 업데이트(MOD-460/461), retriever 인덱스 갱신(MOD-462)
    
- **중단 조건**: 저장 실패 시 PARTIAL 처리 + 재시도 정책
    
- **아티팩트**: `db.sqlite` 업데이트, 문서 스토어 업데이트
    

---

### S99 EndIteration

- **입력**: iteration stats
    
- **출력**: iteration summary.json
    
- **아티팩트**: `iteration_summary.json`
    

---

## 3) Iteration 반복 정책(언제 다음 Iteration으로?)

### 3.1 반복 트리거(예시 정책)

- `target_iterations` 미도달 AND
    
- 최근 K회 동안 **best_score가 개선**되거나 OR
    
- failure→constraint로 탐색 공간이 줄면서 **새 후보 생성**이 의미 있다고 판단
    

### 3.2 종료 트리거

- N회 연속 개선 없음(plateau)
    
- 제약이 과도해져 후보 생성량이 임계 이하
    
- DQ CRITICAL 지속 발생(데이터 문제 해결 우선)
    
- 운영 모드에서 Risk Guard 트립(라이브는 즉시 종료)
    

---

## 4) 재시도/에러 처리 정책

- **단계 단위 재시도**(권장):
    
    - 데이터 로드/일시 오류: 재시도 1~3회(백오프)
        
    - 엔진 크래시: 동일 입력으로 1회 재시도 후 FAIL
        
- **전략 단위 실패 격리**:
    
    - 특정 전략만 실패하면 그 전략만 FAILED로 표시하고 전체 iteration은 지속
        
- 모든 실패는:
    
    - error_code + stage_id + strategy_id(가능 시)로 기록
        

---

## 5) 최소 아티팩트 보장 규칙

- iteration이 “성공/실패”와 무관하게 최소 다음은 남아야 함:
    
    - OBS-530(run metadata)
        
    - OBS-540(DQ)
        
    - iteration_summary(카운트/실패율/베스트 전략)
        

---

## 6) 테스트(파이프라인 레벨)

- **TEST-600**: PIPE-200 E2E 1 iteration
    
- **TEST-611**: 동일 seed/data/policy → 동일 strategy_id + 유사 metrics
    
- **TEST-630**: next_bar 정책 위반 탐지(DQ FAIL)
    
- **TEST-651**: replay 가능한 아티팩트 생성 여부(OBS-530/520 필수 존재)
    

---
