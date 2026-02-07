## Artifact Writer & Storage Contract

- **문서 ID**: PIPE-220
    
- **제목**: Artifact Writer & Storage Contract
    
- **버전**: 1.0
    
- **목표**: 파이프라인 실행 결과(전략/메트릭/로그/리포트/분석/제약)를 **일관된 경로·형식·원자성**으로 저장해, 언제든 `run_id`만으로 재현/디버깅/리플레이가 가능하게 한다.
    
- **범위**: 저장 경로 규칙, 파일 포맷(JSON/JSONL), 원자적 저장, resume 규칙, 압축/보관 정책(기본), 인덱싱.
    
- **비범위**: DB 스키마 상세(MOD-460), 리포트 템플릿(MOD-433)
    

---

## 1) 저장 루트 & 경로 규칙

### 1.1 루트 디렉토리

- 기본: `artifacts/runs/<run_id>/`
    
- 모든 산출물은 이 루트 아래에만 저장(상대경로)
    

### 1.2 Iteration 경로

- `artifacts/runs/<run_id>/iterations/<iteration_id>/`
    

### 1.3 표준 서브폴더

- `metrics/` : SCHEMA-340 (탐색 A 결과)
    
- `metrics_verified/` : SCHEMA-340 (검증 B 결과)
    
- `logs/` : OBS-500/510/520
    
- `analysis/` : failures/evidence
    
- `constraints/` : constraint_set 및 이벤트
    
- `reports/` : tearsheet/요약 리포트
    
- `pipeline/` : stage 이벤트, stage 마커
    

---

## 2) 파일 포맷 계약(Formats)

### 2.1 JSON vs JSONL

- **JSON**: 단일 객체(설정/메타/전략 스펙/메트릭)
    
- **JSONL**: 대량 이벤트/로그/리스트(결정/주문체결/트레이드, 후보/쇼트리스트)
    

### 2.2 압축

- 대형 JSONL은 `.jsonl.gz` 허용(권장)
    
- 압축 여부는 run policy로 고정하고 OBS-530에 기록(권장)
    

---

## 3) 필수 아티팩트 목록(Minimum Set)

### 3.1 Run 단위(필수)

- `run_metadata.json` (OBS-530)
    
- `dq_checks.jsonl` (OBS-540) _(DQ gate를 수행한 경우 필수)_
    

### 3.2 Iteration 단위(필수)

- `iteration_summary.json`
    
- 최소 결과:
    
    - `metrics/<strategy_id>.json` 또는 `metrics_verified/<strategy_id>.json` 중 최소 1개
        
    - `logs/trades.jsonl` (OBS-520, 최소)
        

### 3.3 검증(shortlist) 단위(권장 기본)

- `logs/decisions.jsonl` (OBS-500)
    
- `logs/orders_fills.jsonl` (OBS-510)
    
- `logs/trades_verified.jsonl` (OBS-520)
    

---

## 4) 파일명 규칙(Naming)

### 4.1 전략 단위 파일

- 메트릭:
    
    - `metrics/<strategy_id>.json`
        
    - `metrics_verified/<strategy_id>.json`
        
- 리포트:
    
    - `reports/<strategy_id>.md|html|pdf`
        

### 4.2 집합 파일(JSONL)

- 후보/쇼트리스트:
    
    - `candidates.jsonl`, `deduped.jsonl`, `shortlist.jsonl`
        
- 스코어:
    
    - `scores.jsonl`
        
- 분석/제약 이벤트:
    
    - `analysis/failures.jsonl`
        
    - `constraints/events.jsonl`
        

### 4.3 로그 파일(고정 이름)

- `logs/decisions.jsonl`
    
- `logs/orders_fills.jsonl`
    
- `logs/trades.jsonl`
    
- `logs/trades_verified.jsonl`
    

---

## 5) 원자적 저장(Atomic Write)

### 5.1 원칙

- 부분 저장/깨진 파일이 남지 않도록:
    
    - `write to tmp → fsync → rename` 순으로 저장
        
- JSONL은 “append-only”가 기본이지만, stage 단위로 분리 저장을 권장:
    
    - stage 종료 시점에 파일을 닫고 마커 생성
        

### 5.2 tmp 규칙

- 임시 파일은 같은 디렉토리 내:
    
    - `<name>.tmp.<pid>.<uuid>`
        
- rename은 같은 파일시스템에서 수행(원자성 보장)
    

---

## 6) Resume / Stage Marker 규칙

### 6.1 Stage 완료 마커(권장)

- `pipeline/.S50_DONE`, `pipeline/.S70_DONE` 같은 마커 파일 생성
    
- resume 시:
    
    - 마커가 있고 산출물이 존재하면 stage를 스킵(또는 재검증 후 스킵)
        

### 6.2 입력 해시(필수 권장)

- stage 산출물 옆에 `inputs_hash.json` 저장(권장)
    
- resume 시 현재 입력과 해시가 다르면:
    
    - 스킵 금지(재실행 필요) + pipeline 이벤트로 기록
        

---

## 7) 인덱스 파일(Indexing for Retrieval)

대량 산출물을 빠르게 조회하기 위해 iteration에 인덱스를 둔다(권장).

### 7.1 iteration_index.json (권장)

- 포함 예:
    

```json
{
  "run_id": "uuid",
  "iteration_id": 3,
  "counts": { "generated": 2000, "deduped": 1200, "shortlist": 30, "verified": 20 },
  "best": { "strategy_id": "hash", "score": 0.81 },
  "paths": {
    "candidates": "candidates.jsonl",
    "shortlist": "shortlist.jsonl",
    "trades": "logs/trades.jsonl"
  }
}
```

### 7.2 run_index.json (권장)

- iterations 요약, 베스트 히스토리, 실패율 트렌드
    

---

## 8) 보관/정리 정책(Retention & Cleanup)

### 8.1 탐색(A) vs 검증(B)

- A는 양이 크므로:
    
    - full decision/order 로그는 기본 비활성(정책)
        
    - 메트릭 + trade summary 중심 저장
        
- B는 수가 적으므로:
    
    - 풀 로그 장기 보관 권장
        

### 8.2 정리 규칙(권장)

- 실패한 후보의 원본 candidates는 일정 조건에서 삭제 가능(옵션)
    
- 단, 재현성에 필요한 최소 셋(OBS-530/540 + shortlist/B 결과)은 보존
    

---

## 9) 에러/예외 처리

- `ARTIFACT_WRITE_FAIL`: 디스크/권한 문제
    
- `ARTIFACT_SCHEMA_INVALID`: validator 실패(저장 금지)
    
- `ARTIFACT_PATH_CONFLICT`: 동일 파일명 충돌(정책: overwrite 금지 권장)
    
- `ARTIFACT_ATOMIC_RENAME_FAIL`: 원자적 rename 실패(파일시스템 이슈)
    

---

## 10) 테스트 요구사항

- **TEST-660**: tmp→rename 원자성(중단/크래시 후 깨진 파일 없음)
    
- **TEST-661**: resume 시 stage 마커가 있으면 스킵되고 결과가 동일
    
- **TEST-662**: schema invalid 데이터는 저장되지 않음
    
- **TEST-663**: index 파일이 실제 파일 경로와 정합
    

---
