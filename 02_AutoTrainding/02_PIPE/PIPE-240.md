## Promotion & Governance Pipeline

- **문서 ID**: PIPE-240
    
- **제목**: Promotion & Governance Pipeline
    
- **버전**: 1.0
    
- **목표**: 검증(Engine B)까지 통과한 전략을 “좋아 보인다”에서 끝내지 않고, **승격(Promotion)** 기준에 따라 템플릿/룰/제약/정책/리포트를 체계적으로 관리해 **재현 가능·감사 가능·회귀 가능**한 운영 상태로 올린다.
    
- **범위**: 승격 단계, 티어(등급) 정의, 게이트, 버전/승인 정책, 회귀 테스트 연결, KB 반영 규칙.
    
- **비범위**: 리포트 렌더링(MOD-433), 라이브 OMS 운영(MOD-481/482)
    

---

## 1) 승격 티어(Tiers)

전략은 검증 이후 아래 티어 중 하나로 분류된다.

- **T0 Candidate**: Engine A 통과(탐색 단계)
    
- **T1 Verified**: Engine B 검증 통과(OBS 풀로그/SCHEMA-340 확보)
    
- **T2 Robust**: 강건성 테스트(MOD-432) 통과(워크포워드/국면분할/민감도)
    
- **T3 Paper Ready**: 페이퍼 트레이딩 운영 기준 충족(안전장치/실행 품질)
    
- **T4 Live Ready**: 라이브 운영 기준 충족(거버넌스/리스크/감사 완비)
    
- **T5 Core Template**: 재사용 가능한 템플릿/룰 패밀리로 승격(MOD-413 라이브러리 반영)
    

> 기본 MVP 목표: 최소 T1까지 자동화, T2~는 단계적 확장.

---

## 2) 승격 단계(Stage Flow)

### P10 Verify Complete → P20 Robustness → P30 Paper → P40 Live → P50 Template/Core

1. **P10 Verification Completion**
    

- 입력: Engine B metrics + OBS-500/510/520 + OBS-530/540
    
- 출력: `promotion_record`(승격 후보 레코드), T1 여부
    

2. **P20 Robustness Gate (옵션, MOD-432)**
    

- 입력: 워크포워드/국면분할/민감도 결과
    
- 출력: T2 여부, 취약 구간 요약
    

3. **P30 Paper Readiness Gate**
    

- 입력: 실행 품질/슬리피지/거절률, 주문 빈도/턴오버, 리스크 정책 적합성
    
- 출력: T3 여부, 운영 파라미터(리밸런싱/캡)
    

4. **P40 Live Readiness Gate**
    

- 입력: SYS-130 기준 충족(키/권한/멱등성/리스크가드/감사)
    
- 출력: T4 여부
    

5. **P50 Template/Core Promotion (옵션)**
    

- 입력: 재사용 가치 평가(구조적 패턴/성능 범용성/제약과의 호환)
    
- 출력: 템플릿/원자 룰 라이브러리(MOD-413) 반영, T5 여부
    

---

## 3) 승격 게이트(Gates)

### 3.1 T1 (Verified) 최소 게이트(권장)

- DQ CRITICAL 없음(OBS-540)
    
- 검증 엔진 B 정상 완료(OBS-500/510/520 존재)
    
- 최소 거래 수 `min_trades` 충족
    
- max drawdown 바닥선 통과
    
- 실행 품질:
    
    - reject_rate ≤ 임계
        
    - slippage_p95 ≤ 임계(자산군별)
        

### 3.2 T2 (Robust) 게이트(예시)

- 워크포워드 OOS 유지율 임계 이상
    
- 레짐 슬라이스에서 “한쪽만 잘되는” 편향이 과도하지 않음
    
- 파라미터 민감도(근방 변화)에 성과 급락이 없을 것
    

### 3.3 T3 (Paper Ready) 게이트(예시)

- 평균 주문/일 ≤ 임계(운영 가능성)
    
- 턴오버/비용 비율 ≤ 임계
    
- 리스크 가드 정책(손실 한도/중단 조건) 설정 완료
    

### 3.4 T4 (Live Ready) 게이트(필수)

- SYS-130 준수:
    
    - secrets 마스킹, 권한 분리, idempotency, risk guard, 감사 로그
        
- 브로커 어댑터/OMS 필수 플로우 통과(MOD-480/481)
    

---

## 4) Promotion Record(정형 저장)

승격 결정을 “나중에” 추적하려면 레코드를 고정해야 한다.

### 4.1 저장 위치(권장)

- MOD-460 Local DB: `promotion_records` 테이블(정형)
    
- artifacts:
    
    - `iterations/<id>/promotion/promotion_<strategy_id>.json`
        

### 4.2 필수 필드(권장)

```json
{
  "run_id": "uuid",
  "iteration_id": 7,
  "strategy_id": "hash",
  "from_tier": "T0",
  "to_tier": "T1",
  "decision": "PROMOTE|HOLD|REJECT",
  "score_total": 0.78,
  "gates": {
    "dq_pass": true,
    "min_trades_pass": true,
    "max_dd_pass": true,
    "exec_quality_pass": true
  },
  "evidence_refs": {
    "metrics_verified": "metrics_verified/hash.json",
    "obs_decisions": "logs/decisions.jsonl",
    "obs_orders_fills": "logs/orders_fills.jsonl",
    "obs_trades": "logs/trades_verified.jsonl"
  },
  "policy_versions": { "score": "score_v1", "exec": "exec_v1", "cost": "cost_v1", "pf": "pf_v1" },
  "timestamp": "2026-02-06T00:00:00Z"
}
```

---

## 5) 버전/승인 정책(Governance)

### 5.1 자동 vs 수동 승인

- T1은 자동 승격 가능(정책 기반)
    
- T3/T4는 기본 “수동 승인”을 권장(운영 위험 때문)
    
- 승인 이벤트는 promotion record에 `approved_by`, `approved_at`(옵션)로 남김
    

### 5.2 정책 고정(중요)

- 동일 run/iteration에서 승격 기준(게이트/임계값)은 **고정**
    
- 변경이 필요하면:
    
    - 새 policy_version으로 새로운 run에서 재검증(재현성 유지)
        

---

## 6) 회귀 테스트(Regression) 연결

승격된 전략/템플릿은 회귀 테스트 대상이 된다.

- T1 이상:
    
    - TEST-600/610/650 기본 통과 확인
        
- T2 이상:
    
    - robustness 회귀(워크포워드/레짐) 샘플 1회 재실행
        
- T4:
    
    - paper→live 모의 시나리오 테스트(TEST-671/672 연동)
        

---

## 7) Knowledge Base 반영 규칙(MOD-460~462)

- T1 이상 전략:
    
    - 전략 스펙 + 검증 메트릭 + 실패/제약 히스토리 저장
        
- T5(Core Template) 승격 시:
    
    - MOD-413 템플릿 라이브러리에 등록
        
    - “적용 조건/금지 조건”을 constraint 형태로 같이 저장(안전장치)
        

---

## 8) 실패/보류 처리(Hold/Reject)

- HOLD:
    
    - 성능은 좋지만 실행/비용/리스크 조건이 미달 → 제약/파라미터 조정 대상으로 유지
        
- REJECT:
    
    - DQ CRITICAL, 과최적화 명확, 실행 품질 치명적 → 후보 풀에서 제외
        
- 모든 HOLD/REJECT에도 근거(evidence_refs)를 남김(나중에 왜 버렸는지 재현 가능)
    

---

## 9) 예외/에러 처리

- `PROMOTION_EVIDENCE_MISSING`: 필수 근거 파일 누락
    
- `PROMOTION_POLICY_MISMATCH`: 승격 판단 당시 정책 버전 불일치
    
- `PROMOTION_REGRESSION_FAIL`: 승격 후 회귀 테스트 실패
    

---

## 10) 테스트 요구사항

- **TEST-690**: T1 승격 시 promotion_record 생성 + evidence_refs 유효
    
- **TEST-691**: policy_version 변경 없이 임계값 변경 시도 → 차단(거버넌스)
    
- **TEST-692**: 승격된 전략이 회귀 테스트 스위트에 자동 등록(리스트/태그)
    
- **TEST-693**: evidence 누락 시 HOLD/FAIL 처리 + 이벤트 기록
    

---
