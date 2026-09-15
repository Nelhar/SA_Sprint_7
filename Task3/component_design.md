# Дизайн компонентов: API, Database, Cache

**Инициатива:** #1 Выявление подозрительных операций  

---

## Rules Engine Component

**Вход:** Operation (account_id, amount, counterparty_id, memo)  
**Выход:** Score A (0-1), Action (ALLOW/SKIP/BLOCK)

```python
def rules_engine(operation):
    if operation.amount < 30_000 and is_whitelisted(operation.counterparty_id):
        return 0.0, "AUTO_ALLOW"  # Microfraud, low risk
    if operation.amount > 1_000_000 and is_foreign(operation.counterparty_id):
        return None, "SKIP_TO_ML"  # Requires ML analysis
    if is_blacklisted_ofac(operation.counterparty_id):
        return 1.0, "AUTO_BLOCK"  # Regulator requirement
    return None, "SKIP_TO_ML"
```

---

## ML Service Component

**Вход:** Features (50+ признаки)  
**Выход:** Score B (0-1), Top-5 SHAP values

```python
import lightgbm as lgb
import joblib

model = joblib.load("fraud_detection_v1.0.pkl")

def ml_service(features_dict):
    X = np.array([features_dict[key] for key in FEATURE_ORDER])
    score = model.predict(X)[0]
    shap_values = explainer.shap_values(X)[0]
    return {
        "score": float(score),
        "top_5_factors": sorted([(FEATURE_ORDER[i], shap_values[i]) 
                                  for i in np.argsort(np.abs(shap_values))[-5:]],
                                 key=lambda x: x[1], reverse=True)
    }
```

---

## Decision Dispatcher Component

**Вход:** Operation, Rules Score A, ML Score B  
**Выход:** Decision (AUTO_ALLOW, EXPERT_REVIEW, AUTO_BLOCK) + Audit log

```python
def dispatcher(operation, score_a, score_b):
    if score_a is not None:
        return score_a, "AUTO_DECIDED_BY_RULES"
    
    if score_b < 0.3:
        return 0.0, "AUTO_ALLOW"
    elif score_b < 0.7:
        return score_b, "EXPERT_REVIEW"
    else:
        return 1.0, "AUTO_BLOCK"
```

---

## Database Schema

```sql
CREATE TABLE operations (
    id BIGINT PRIMARY KEY,
    account_id BIGINT,
    amount DECIMAL,
    counterparty_id BIGINT,
    memo VARCHAR(500),
    created_at TIMESTAMP,
    -- ML columns
    rules_score FLOAT,
    ml_score FLOAT,
    decision VARCHAR(20),  -- AUTO_ALLOW, EXPERT_REVIEW, AUTO_BLOCK
    expert_decision VARCHAR(20),  -- NULL, ALLOW, BLOCK, INVESTIGATE
    expert_id INT,
    decision_time TIMESTAMP,
    INDEX (created_at),
    INDEX (account_id, created_at)
);

CREATE TABLE ml_audit_log (
    id BIGINT PRIMARY KEY,
    operation_id BIGINT,
    model_version VARCHAR(10),
    inference_time_ms FLOAT,
    features_hash VARCHAR(64),
    created_at TIMESTAMP,
    FOREIGN KEY (operation_id) REFERENCES operations(id)
);
```

---

## Cache Strategy (Redis)

```python
def get_features_cached(operation):
    key = f"features:{operation.account_id}:{operation.created_at.date()}"
    
    # Try cache (60s TTL)
    cached = redis.get(key)
    if cached:
        return json.loads(cached)
    
    # Cache miss → query DB (20ms)
    features = compute_features(operation)
    redis.setex(key, 60, json.dumps(features))
    return features
```

---

## API Contract (OpenAPI)

```yaml
POST /fraud/score
Request:
  {
    "operation": {
      "account_id": 12345,
      "amount": 50000,
      "counterparty_id": 67890,
      "memo": "Payment for services",
      "timestamp": "2024-10-16T10:30:00Z"
    }
  }

Response (200):
  {
    "decision": "EXPERT_REVIEW",
    "score": 0.62,
    "latency_ms": 45,
    "model_version": "v1.0",
    "factors": [
      {"name": "foreign_country", "impact": 0.32},
      {"name": "amount_deviation", "impact": 0.24}
    ]
  }

Response (503):
  {
    "decision": "FALLBACK_TO_RULES",
    "reason": "ML service timeout"
  }
```

---

## Deployment Topology

```
┌─────────────────┐
│  API Gateway    │
│  (Kong)         │
└────────┬────────┘
         │
    ┌────┴────┐
    │          │
┌───▼──┐   ┌──▼───┐
│ Req1 │   │ Req2 │  (Load balancing)
└───┬──┘   └──┬───┘
    │         │
┌───▼─────────▼───┐
│ ML Service Pool │
│  (Python+ASGI) │
│  3 replicas     │
└───┬─────────────┘
    │
┌───▼──────────────┐
│ Shared Resources │
├──────────────────┤
│ LightGBM model   │
│ Redis (cache)    │
│ Postgres (DB)    │
└──────────────────┘
```

---

## Error Handling

```
Operation → ML Service
  │
  ├─ Success (200ms) → Use score
  │
  ├─ Timeout (200ms) → Fallback to RULES
  │   └─ If rules also fail → Queue to MANUAL_REVIEW
  │
  ├─ 4xx Error → Log, alert, QUEUE TO MANUAL_REVIEW
  │
  └─ 5xx Error → Circuit breaker (stop calling for 30s)
      └─ Fallback to RULES
```

---

**Next:** Fallback Strategy, Monitoring Plan, ADR Architecture.
