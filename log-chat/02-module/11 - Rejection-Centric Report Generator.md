(기각 중심 최종 보고서 생성 단계)

**목적:**  
10단계에서 확정된 **단일 시나리오 판결**을  
→ **근거·과정·기각 사유·불확실성까지 포함한 최종 보고서**로 변환한다.

> 핵심 철학
> 
> - 이 보고서는 “설득”이 아니라 **검증 가능성**을 목표로 한다
>     
> - 무엇을 선택했는지보다 **무엇을 버렸는지**가 더 중요하다
>     
> - 읽는 사람이 _동의하지 않아도_ **이해는 가능**해야 한다
>     

---

## 1) 입력과 출력

### 입력 (Final Decision Pack)

10단계 출력 전체

`{   "final_scenario": {...},   "decision_log": {...} }`

---

### 출력 (Final Report)

사람이 읽는 **최종 산출물**  
(문서 / PDF / Markdown / UI 렌더링 등)

`{   "report": {     "executive_summary": "...",     "final_scenario": {...},     "evidence_summary": [...],     "timeline_overview": [...],     "rejected_scenarios": [...],     "remaining_uncertainties": [...],     "falsification_checklist": [...],     "confidence_statement": {...},     "appendix": {       "evidence_cards": [...],       "decision_log": {...},       "failure_ontology_updates": [...]     }   } }`

---

## 2) 보고서 전체 구조 (고정 템플릿)

### 2-1. Executive Summary (1페이지 요약)

**질문에 바로 답한다.**

- 사용자가 던진 주제 요약
    
- **최종 시나리오 1개 (한 문단)**
    
- 신뢰도(정량 + 정성)
    
- 핵심 전제 1~2개
    
- 가장 큰 리스크 1개
    

> ⚠️ 여기서는 **판결만** 제시  
> 근거·논쟁은 아래에서

---

### 2-2. Final Scenario (단일 시나리오 본문)

다음 순서로 서술한다:

1. **Observed Facts (관측된 사실)**
    
    - Evidence ID 포함
        
    - 평가/의미 부여 금지
        
2. **Interpretation (해석)**
    
    - 사실을 어떻게 연결했는지
        
    - 인과 단정 표현 최소화
        
3. **Assumptions (가정)**
    
    - 남아 있는 가정
        
    - 왜 필요한지
        
    - 비용(리스크)
        

---

### 2-3. Evidence Summary (근거 요약)

- 핵심 근거 카드 5~10개
    
- 출처 등급(Tier) 명시
    
- 서로 독립적인지 표시
    

> “이 결론은 어떤 자료 위에 서 있는가?”

---

### 2-4. Timeline Overview (사건 흐름)

- 핵심 사건을 시간순으로 나열
    
- 인과 해석 없이 **사실 서술만**
    

---

### 2-5. Rejected Scenarios (기각된 시나리오들)

이 보고서의 **가장 중요한 파트**

각 시나리오마다:

- 한 줄 요약
    
- 왜 그럴듯했는지
    
- **왜 기각되었는지**
    
    - 논리
        
    - 전제 신뢰성
        
    - 현실 충돌
        
- 어떤 evidence가 기각을 촉발했는지
    

> 독자가 “왜 이건 안 되는지”를 이해해야  
> 최종 선택도 신뢰한다

---

### 2-6. Remaining Uncertainties (남은 불확실성)

- 해결되지 않은 pivotal assumption
    
- 그 영향도(high/medium/low)
    
- 추가로 확인되면 **결론이 바뀔 수 있는지**
    

> 불확실성을 숨기지 않는다  
> **관리 대상으로 명시**

---

### 2-7. Falsification Checklist (반증 체크리스트)

**이게 나오면 결론을 수정해야 한다**

예:

- 새로운 공식 문서 A 공개
    
- 특정 수치가 X 이하로 수정
    
- 결정적 사건의 날짜 변경
    

→ 이 리스트가 있는 한,  
보고서는 **열린 상태**를 유지한다

---

### 2-8. Confidence Statement

- confidence_score (0~1)
    
- confidence_label (low/medium/high)
    
- 왜 이 점수인지 설명
    

---

### 2-9. Appendix (감사/재현용)

- 사용된 Evidence Card 전체
    
- Decision log (어떤 규칙이 적용됐는지)
    
- Failure Ontology 업데이트 항목
    

---

## 3) 에이전트 책임과 금지사항

### Report Generator Agent 책임

- 판결 결과를 **정확히 반영**
    
- 구조화된 보고서 생성
    
- 기각 사유 명확화
    

### 금지사항

- 판결을 미화하거나 축소
    
- 기각 이유를 흐리게 표현
    
- 새로운 해석 추가
    

---

## 4) 품질 게이트 (보고서 합격 기준)

보고서는 다음을 만족해야 한다:

1. Final Scenario는 1개
    
2. Rejected Scenario ≥ 1
    
3. Evidence ID가 모든 핵심 주장에 연결됨
    
4. 반증 체크리스트가 존재
    
5. 불확실성이 명시됨
    

---

## 5) 운영 지표

- Reader agreement vs understanding (동의보다 이해)
    
- Rejection clarity score
    
- Evidence traceability
    
- Update-trigger rate (반증 발생 시)
    

---

## 6) 이 단계의 결정적 의미

- 시스템 결과가 **블랙박스가 아님**
    
- 외부 감사/반박/업데이트 가능
    
- “AI가 말했다”가 아니라  
    **“이 근거 때문에 이렇게 판결됐다”**가 된다
    

---

# 전체 파이프라인 완결

1️⃣ Topic & Claim Decomposer  
2️⃣ Data Requirements Spec  
3️⃣ Evidence Collection & Curation  
4️⃣ Evidence Map / Timeline  
5️⃣ Evidence-Constrained Scenario Generation  
6️⃣ Pivotal Assumption Finder  
7️⃣ Targeted Retrieval Loop  
8️⃣ Cross-Critique  
9️⃣ 3-Layer Deductive Verification  
🔟 Judge / Convergence  
1️⃣1️⃣ Rejection-Centric Report Generator

---

## 마지막 한 문장 정리

이 시스템은  
**“가능성을 나열하는 AI”가 아니라  
“근거 위에서 판결을 내리고, 틀릴 준비가 된 시스템”**입니다.