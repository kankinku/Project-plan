# INDEX — Auto Trading Strategy Agent Docs

## 0. 문서 규칙(Documentation Conventions)

- [[DOC-000]] 문서 작성 규칙(네이밍, 버전, 변경로그, 용어)
    
- [[DOC-001]] 용어집(Glossary) / 약어 / 표준 단위(시간/수익률/비용)
    
- [[DOC-002]] 공통 데이터 타입(JSON/AST) 규약(스키마 버전, 타입, 에러 포맷)
    
- [[DOC-003]] Reason Codes 표준(네이밍/심각도/권장 액션)
    

---

## 1. 제품/시스템 개요(Top-level)

- [[SYS-100]] 시스템 개요(목표, 범위, 성공 기준, MVP/V1/V2)
    
- [[SYS-110]] 전체 아키텍처(모듈 관계도, 데이터 흐름, 실행 흐름)
    
- [[SYS-120]] 실행 모드 정의(backtest/paper/live, dry-run, replay)
    
- [[SYS-130]] 보안/키관리/권한(브로커 API, 데이터 소스, 로그 마스킹)
    

---

## 2. 파이프라인 단계(End-to-End Stages)

> “무한 반복 루프”를 문서화 단위로 분해한 섹션

- [[PIPE-200]] 반복 루프(Orchestrator) 상태 머신/그래프 정의
    
- [[PIPE-210]] Stage 1 — 전략 생성(Generator)
    
- [[PIPE-220]] Stage 2 — 백테스트(Backtester: 탐색용/검증용)
    
- [[PIPE-230]] Stage 3 — 평가/스코어링(Scorer)
    
- [[PIPE-240]] Stage 4 — 실패 원인 분석(Analyzer)
    
- [[PIPE-250]] Stage 5 — 제약 생성/탐색 축소(Constraint Writer)
    
- [[PIPE-260]] Stage 6 — 지식 축적/회상(KB Writer & Retriever)
    
- [[PIPE-270]] Stage 7 — 사용자 시나리오 입력→JSON→반영(Scenario Intake) _(V2)_
    

---

## 3. 핵심 도메인 스키마(DSL/AST/시나리오)

- [[SCHEMA-300]] Strategy DSL 스펙(전체 JSON 구조)
    
- [[SCHEMA-310]] Condition AST 스펙(연산자, 타입체크, 정규화, 해시)
    
- [[SCHEMA-320]] Scenario JSON 스키마(자연어→구조화/수정 제안)
    
- [[SCHEMA-330]] Constraints 스키마(BAN/LIMIT/PRIORITY/PORTFOLIO/EXECUTION)
    
- [[SCHEMA-340]] Metrics 스키마(표준 지표명, 단위, 계산 규칙)
    

---

## 4. 모듈별 상세 문서(Interface & Behavior)

> “모듈=단위 기능” 문서들. 각 문서는 **입력/출력/상태/로그/예외/테스트** 템플릿으로 작성.

### 4.1 Orchestrator

- [[MOD-400]] Orchestrator(상태 정의, 재시도, 중단 조건, 분기)
    
- [[MOD-401]] Run/Iteration/Strategy ID 정책(해시/버전/재현성)
    

### 4.2 Strategy Generator

- [[MOD-410]] Generator(템플릿 기반/원자 규칙 기반/LLM 기반)
    
- [[MOD-411]] Parameter Sampler(범위, 분포, 시드, 민감도 설정)
    
- [[MOD-412]] Strategy Normalizer & Hash(동치성, 정렬, 중복 제거)
    
- [[MOD-413]] Rule Template Library(자체 부트스트랩 저장/승격 규칙)
    

### 4.3 Backtester

- [[MOD-420]] Backtester 공통 인터페이스(엔진 추상화)
    
- [[MOD-421]] Backtest Engine A(탐색: vectorized)
    
- [[MOD-422]] Backtest Engine B(검증: event-driven)
    
- [[MOD-423]] Cost Model(수수료/슬리피지/스프레드)
    
- [[MOD-424]] Execution Model(체결 규칙, 지연, ADV 제한)
    
- [[MOD-425]] Portfolio Model(포지션/현금/레버리지/마진)
    

### 4.4 Scorer & Report

- [[MOD-430]] Metrics Calculator(지표 정의/연율화/리샘플링)
    
- [[MOD-431]] Scorer(가중치, 페널티, OOS 유지율)
    
- [[MOD-432]] Robustness Tests(워크포워드/국면분할/민감도)
    
- [[MOD-433]] Report Generator(티어시트/아티팩트/요약)
    

### 4.5 Root-cause Analyzer

- [[MOD-440]] Failure Taxonomy(카테고리/태그/증거 규격)
    
- [[MOD-441]] Analyzer Rules v0(룰 기반 진단 로직)
    
- [[MOD-442]] Evidence Builder(어떤 로그/구간/지표를 근거로 남기는가)
    

### 4.6 Constraints

- [[MOD-450]] Constraint Writer(실패→제약 변환 규칙)
    
- [[MOD-451]] Constraint Applicator(Generator/Portfolio/Execution에 주입)
    
- [[MOD-452]] Constraint Lifecycle(활성/비활성/만료/회귀 테스트)
    

### 4.7 Knowledge Base

- [[MOD-460]] Local DB(정형: 전략/결과/실패/제약)
    
- [[MOD-461]] Document Store(비정형: 리포트/사유/시나리오)
    
- [[MOD-462]] Retriever(유사 실패/유사 전략 검색)
    

### 4.8 Scenario Intake (V2)

- [[MOD-470]] Scenario Parser(NL→JSON)
    
- [[MOD-471]] Scenario Quality Scoring(명확성/검증가능성/일관성)
    
- [[MOD-472]] Scenario → Strategy Integration(soft 반영/스트레스 테스트)
    

### 4.9 Live Trading

- [[MOD-480]] Broker Adapter(주문/체결/상태)
    
- [[MOD-481]] OMS(주문관리, 재시도, idempotency)
    
- [[MOD-482]] Risk Guard(거래중단, 손실 한도, 이상징후 탐지)
    
- [[MOD-483]] Replay Engine(실시간 로그 기반 리플레이)
    

---

## 5. 로그/데이터/DB(Observability & Reproducibility)

- [[OBS-500]] Decision Log 스펙(bar 단위)
    
- [[OBS-510]] Order & Fill Log 스펙
    
- [[OBS-520]] Trade Summary 스펙(MFE/MAE 포함)
    
- [[OBS-530]] Run Metadata(버전, 시드, 설정 고정)
    
- [[OBS-540]] Data Quality Checks(룩어헤드/서바이버십/타임존 등)
    

---

## 6. 테스트/검증(Testing & QA)

- [[TEST-600]] 단위 테스트 가이드(AST/스키마/지표 계산)
    
- [[TEST-610]] 통합 테스트(파이프라인 E2E)
    
- [[TEST-620]] 재현성 테스트(동일 입력→동일 출력)
    
- [[TEST-630]] 백테스트 신뢰성 테스트(룩어헤드/체결/비용)
    
- [[TEST-640]] 회귀 테스트(제약 추가 후 성능 악화 감지)
    

---

## 7. 운영/배포(Ops)

- [[OPS-700]] 배포 구조(dev/stage/prod)
    
- [[OPS-710]] 스케줄링/잡 관리(배치/실시간)
    
- [[OPS-720]] 모니터링/알람(손실/체결/지연/장애)
    
- [[OPS-730]] 장애 대응 Runbook(브로커/데이터/API)
    

---

## 8. 로드맵/변경 관리

- [[PLAN-800]] 마일스톤(MVP/V1/V2) 상세
    
- [[PLAN-810]] 기술 부채/리스크 레지스터
    
- [[PLAN-820]] 버전 정책(스키마/엔진/지표/전략)
    

---

# 부록(Templates)

- [[TPL-900]] 모듈 문서 템플릿(입력/출력/상태/로그/예외/테스트)
    
- [[TPL-910]] API/JSON 스키마 템플릿
    
- [[TPL-920]] Reason code 추가 템플릿
    
- [[TPL-930]] 데이터 품질 체크 템플릿
    

---

