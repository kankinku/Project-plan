(단일 시나리오 판결 단계)

**목적:**  
9단계의 연역 검증 결과를 바탕으로  
→ **후보 시나리오 중 단 하나를 최종 시나리오로 판결**한다.  
→ 불확실성이 남아도 **판결은 반드시 내려야 하며**,  
→ 남은 리스크는 **명시적으로 드러낸다**.

> 핵심 포인트
> 
> - 이 단계만이 **선택 권한**을 가진다
>     
> - 새 시나리오 생성 ❌
>     
> - “가장 덜 틀린 하나”를 고른다
>     

---

## 1) 입력과 출력

### 입력 (Verification Pack)

9단계 출력 전체

`{   "verification_results": [...],   "scenario_vulnerability": [...],   "consensus_signals": [...],   "resolution_status": [...],   "unresolved_assumptions": [...] }`

---

### 출력 (Final Decision Pack)

이 출력은 **11단계(Report Generator)**의 입력이 된다.

`{   "final_scenario": {     "scenario_id": "S3",     "decision_status": "selected|selected_with_reservations",     "confidence_score": 0.73,     "confidence_label": "medium",     "why_selected": [       "논리 검증 통과",       "결정적 전제 A1 해결",       "타임라인 일관성 유지",       "다른 시나리오 대비 가정 수 최소"     ],     "key_evidence_ids": ["E1", "E7", "E9"],     "remaining_uncertainties": [       {         "assumption_id": "A2",         "impact": "low",         "description": "보조 메커니즘 미확인"       }     ],     "rejected_scenarios": [       {         "scenario_id": "S1",         "fatal_reason": "logical_invalidity"       },       {         "scenario_id": "S2",         "fatal_reason": "premise_unreliable"       }     ]   },   "decision_log": {     "rules_applied": [       "logical_gate",       "pivotal_resolution_priority",       "assumption_penalty"     ],     "tie_breakers_used": [       "evidence_quality",       "assumption_count",       "reality_consistency"     ]   } }`

---

## 2) 판결 규칙 체계 (Decision Policy)

### 2-1. Gate Rules (탈락 규칙)

다음 중 하나라도 해당하면 **자동 탈락**:

- Logical Validity = fail
    
- Reality Consistency = fail
    
- 치명적 pivotal assumption fail
    

---

### 2-2. Scoring Dimensions (정량 평가)

탈락되지 않은 시나리오에 대해 점수 계산:

|항목|설명|
|---|---|
|Logic Score|논리 검증 결과|
|Evidence Quality|T1/T2 비중, 독립성|
|Assumption Load|가정 수 + 비용|
|Pivotal Resolution|결정적 전제 해결 여부|
|Vulnerability|Cross-Critique 취약성|
|Consensus Bonus|다른 시나리오와 공통 근거|

> **가정이 많을수록 자동 감점**

---

### 2-3. Tie-breaker (동점 시 규칙)

점수가 유사할 경우 아래 순서로 결정:

1. **검증 가능한 시나리오**
    
2. **가정이 적은 시나리오**
    
3. **타임라인·현실 제약과 더 잘 맞는 시나리오**
    
4. **반증 포인트가 명확한 시나리오**
    

---

### 2-4. 전원 Fail 상황 처리 (Hard Case)

모든 시나리오가 fail인 경우에도:

- **가장 적은 위반**을 가진 시나리오 1개를
    
- `selected_with_reservations`로 강제 선택
    
- 보고서 상단에 **강한 경고 표시**
    

> “판결 회피”는 허용되지 않는다.

---

## 3) 에이전트 책임과 금지사항

### Judge / Convergence Agent 책임

- 판결 규칙 적용
    
- 단일 시나리오 선택
    
- 잔여 불확실성 명시
    

### 금지사항

- 시나리오 수정/재작성
    
- 새로운 근거 추가
    
- “더 조사 필요”로 판결 미루기
    

---

## 4) 품질 게이트 (이 단계의 합격 조건)

항상 합격한다.  
(판결을 내리지 않으면 시스템 실패)

단, `selected_with_reservations` 상태일 경우:

- 경고 레벨 상승
    
- confidence 상한 제한
    

---

## 5) 운영 지표

- Decisiveness rate (항상 100%여야 정상)
    
- Selected_with_reservations 비율
    
- Avg confidence score
    
- Unresolved assumption count
    

---

## 6) 이 단계의 결정적 의미

- 시스템이 **분석 도구를 넘어 ‘의사결정 시스템’이 됨**
    
- 사용자는 “가능성 나열”이 아니라 **판결을 받음**
    
- 남은 불확실성은 숨기지 않고 **관리 대상으로 전환**