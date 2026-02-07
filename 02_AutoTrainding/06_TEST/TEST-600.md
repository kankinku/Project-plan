## Test Harness & QA Baseline (테스트 하네스/품질 기준)

- **문서 ID**: TEST-600
    
- **제목**: Test Harness & QA Baseline Specification
    
- **버전**: 1.0
    
- **목표**: 이 프로젝트(전략 생성→백테스트→스코어링→분석→제약)의 신뢰성을 확보하기 위해, **테스트 계층/규칙/데이터 픽스처/결정성/리포팅**을 표준화한다.
    
- **범위**: 테스트 분류(단위/통합/E2E/회귀), 공통 하네스, 픽스처, 시드/결정성, CI 게이트, 결과 아티팩트.
    
- **비범위**: 개별 모듈별 구체 테스트 케이스 정의(=TEST-610~640)
    

---

## 1) 입력(Inputs)

### 1.1 테스트 설정

- `test_config.yaml/json`
    
    - 엔진 모드(A/B), 데이터 버전, 기간, 시드, 허용 오차(tolerance), 실행 예산(타임아웃)
        
- `ci_profile`
    
    - quick(PR) / full(nightly) / release(태깅) 등
        

### 1.2 테스트 데이터/픽스처

- **미니 데이터셋**: 20~200개 캔들(소형, 결정성 높은 샘플)
    
- **표준 유니버스**: 5~50 심볼(소형), 섹터/유동성 다양성 포함
    
- **정답(골든) 아티팩트**:
    
    - SCHEMA/OBS 샘플 JSON
        
    - 백테스트 결과 요약(특정 전략의 기대 metrics 범위)
        

---

## 2) 출력(Outputs)

- `test_report.json` (머신리더블)
    
    - pass/fail, duration, flaky 여부, 실패 원인 요약, 링크(artifact path)
        
- `junit.xml` (CI 표준 리포트)
    
- 실패 시 **증거팩**:
    
    - 실패한 테스트 케이스의 입력 스냅샷
        
    - 비교(diff) 결과(예: schema mismatch)
        
    - 관련 로그 샘플(OBS-500/510 등)
        

---

## 3) 상태(State)

### 3.1 TestRunState

- `PENDING` → `RUNNING` → `PASSED | FAILED | FLAKY | SKIPPED`
    
- `FLAKY`는 **재시도 후 통과**했지만 안정성이 낮은 상태(정책상 release 차단 가능)
    

### 3.2 Determinism State

- `DETERMINISTIC_OK`
    
- `NON_DETERMINISTIC_DETECTED` (시드 고정인데 결과 변화)
    
    - 즉시 “회귀 테스트” 대상으로 승격
        

---

## 4) 로깅(Logs)

### 4.1 공통 로그

- 테스트 시작/종료, seed, data_version, engine_version, policy_versions 기록
    
- 실패 시:
    
    - (1) assertion 메시지
        
    - (2) 관련 아티팩트 경로
        
    - (3) 최소 재현 커맨드(문자열)
        

### 4.2 성능 로그(권장)

- 주요 단계별 duration:
    
    - generation/backtest/scoring/analysis
        
- 타임아웃 발생 시 “어느 단계에서” 멈췄는지 기록
    

---

## 5) 예외/에러 처리(Exceptions)

### 5.1 Flaky 규정

- 네트워크/외부 의존이 있는 테스트는 원칙적으로 금지
    
- 불가피하면:
    
    - 최대 2회 재시도
        
    - 통과해도 `FLAKY`로 마킹
        
    - nightly에서 안정화될 때까지 release 차단 가능
        

### 5.2 허용 오차(tolerance)

- 수치 비교는 절대값/상대값 tolerance를 사용(예: bps, pct)
    
- 단, **해시/ID/스키마/상태머신 전이**는 tolerance 없이 엄격 비교
    

---

## 6) 테스트 계층(Test Layers)

### 6.1 단위(Unit)

- 순수 함수/스키마 밸리데이터/정규화/해시
    
- 외부 I/O 없이 실행, 빠름(초~수십초)
    

### 6.2 통합(Integration)

- 모듈 간 연결(Generator→Backtest→Score, Analyzer→Constraint 등)
    
- 작은 데이터셋으로 실행, 결정성 확보
    

### 6.3 E2E(End-to-End)

- Orchestrator 1~3 iteration을 실제로 돌려
    
    - artifacts 생성
        
    - failures/constraints 저장
        
    - retriever 인덱스 업데이트(옵션)
        
- “전체 파이프라인이 깨지지 않음”을 확인
    

### 6.4 회귀(Regression)

- 과거에 깨졌던 버그/실패를 “재현용 케이스”로 고정
    
- 특히:
    
    - LOOKAHEAD 방지
        
    - OMS 멱등성
        
    - schema/obs 버전 호환
        

---

## 7) CI 게이트(필수 정책)

### 7.1 PR(빠른 게이트)

- Unit + 핵심 Integration 일부
    
- 스키마 밸리데이터(샘플 JSON) 통과
    
- 결정성 체크(같은 seed 2회 실행 결과 동일)
    

### 7.2 Nightly(전체)

- E2E + robustness 일부 + 성능 스모크
    
- flaky 추적 리포트 생성
    

### 7.3 Release

- Nightly 전부 통과 + flaky=0(또는 허용 리스트만)
    
- NON_DETERMINISTIC_DETECTED=0
    

---

## 8) 테스트 요구사항(이 문서 자체의 Acceptance)

- 테스트 실행 시 **seed/data_version/code_version**이 항상 리포트에 포함된다.
    
- 실패 시 “재현 커맨드 + 입력 스냅샷 경로 + diff”가 남는다.
    
- PR 게이트는 5~10분 내 종료(정책 목표), nightly는 리소스 예산 내.
    
- 동일 입력/시드에서 결과가 바뀌면 즉시 회귀 대상으로 분류된다.
    

---

