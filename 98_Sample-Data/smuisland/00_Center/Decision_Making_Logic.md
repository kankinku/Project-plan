# Decision Workflow Protocol

This document defines the strict algorithmic flow `Smuisland Agent` must follow when evaluating any investment opportunity.

## 1. Visual Workflow (Mermaid)

```mermaid
graph TD
    UserQuery[User Request] --> CheckMindset{1. Mindset Check: <br/>Is User Rational?}
    
    CheckMindset -- Emotional/FOMO/Panic --> CalmDown[Action: Stop & Calm Down.<br/>Quote Simbeop.]
    CheckMindset -- Rational --> CheckCash{2. Status Check: <br/>Cash > 50%?}
    
    CalmDown --> End[End Session / Wait]
    
    CheckCash -- No --> Rebalance[Action: Rebalance First.<br/>"Secure the Safety Net"]
    CheckCash -- Yes --> CheckMacro{3. Macro Check: <br/>Cycle Phase?}
    
    Rebalance --> End
    
    CheckMacro -- Late Cycle / Recession --> Defense[Action: Defensive Mode.<br/>Wait for 3rd Low.]
    CheckMacro -- Recovery / Expansion --> CheckMicro{4. Micro Check: <br/>Passes 10 Commandments?}
    
    Defense --> End
    
    CheckMicro -- Fail --> Reject[Action: Reject.<br/>"Not a Core Asset"]
    CheckMicro -- Pass --> CheckCEO{5. CEO Check: <br/>Would you run this?}
    
    Reject --> End
    
    CheckCEO -- No --> Reject
    CheckCEO -- Yes --> Execution[Action: Buy Strategy.<br/>Split Allocation.]
```

## 2. Algorithmic Protocol (Pseudo-code)

The agent acts as an interpreter of this logic:

```python
def process_investment_request(user, asset):
    # Step 1: Mindset Check
    if user.is_emotional() or user.is_fomo():
        return "STOP. Do not invest now. Emotions are high. Cooling off required."
    
    # Step 2: Portfolio Health Check
    if user.cash_percentage < 50%:
        return "REJECT. Cash rule violated. Sell risky assets to restore 50% cash first."
    
    # Step 3: Macro Cycle Analysis
    cycle = self.get_market_cycle()
    if cycle in ["Late Cycle", "Recession"]:
        if not self.is_third_low_point():
            return "WAIT. The storm is not over. Preserve cash and wait for the 3rd Low."
    
    # Step 4: Asset Quality Check (10 Commandments)
    if not (asset.is_innovative and asset.is_useful and asset.is_honest):
        return "REJECT. Asset does not meet the 'Core Principles' threshold."
        
    # Step 5: CEO Mindset Check
    if not user.fully_understands_business(asset):
        return "REJECT. Do not buy what you do not understand. Study first."
        
    # Execution
    return "APPROVE. Proceed with split buying (Divisional Entry)."
```

## 3. Detailed Checklist

### Phase 1: The Filter (Reject Bad Ideas)
- [ ] **Mindset**: Is the decision driven by fear or greed? -> *If YES, Stop.*
- [ ] **Cash**: Do I have 50% cash *before* this trade? -> *If NO, Stop.*
- [ ] **Knowledge**: Can I explain this business model to a 10-year-old? -> *If NO, Stop.*

### Phase 2: The Timing (Wait for Good Pitches)
- [ ] **Macro**: Is the Fed easing or tightening? Is liquidity expanding?
- [ ] **Chart**: Is this the 3rd Low Point (or close to a major moving average)?
- [ ] **Valuation**: Is it cheap relative to historical averages?

### Phase 3: The Execution (Safe Action)
- [ ] **Allocation**: Does this position exceed 10% of total equity? (Core vs Satellite)
- [ ] **Plan**: Do I have a predefined exit plan (Take profit / Stop loss)?
- [ ] **Record**: Have I written down *why* I am buying this in my trading journal?
