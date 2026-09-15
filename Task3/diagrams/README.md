# Диаграммы архитектуры (C4 Model + PlantUML)

**Инициатива:** #1-3 (Fraud Detection, Delinquency, Churn)

---

## 📊 C4 диаграммы (PlantUML)

### Для флагмана (#1: Fraud Detection)

#### 1. **Container Diagram** 
**Файл:** `c4_container_fraud_detection.puml`

Показывает высокоуровневую архитектуру системы:
- API Gateway (Kong) → Load Balancer
- Rules Engine (30 правил, <2ms)
- ML Service (LightGBM, <5ms)
- Decision Dispatcher (принимает решение)
- Expert Queue (ручная проверка)
- Cache (Redis) + DB (PostgreSQL)
- OFAC Check + Risk DB
- AML Experts (10 человек)
- Monitoring (Prometheus + Grafana)

**Контракт на выходе:**
```json
{
  "decision": "ALLOW|REVIEW|BLOCK|ESCALATE",
  "score": 0-1,
  "confidence": 0-1,
  "reason": "string",
  "shap_top_3": [features],
  "fallback": boolean,
  "trace_id": "uuid"
}
```

**Fallback:** Если ML timeout → fallback на Rules-only decision

**Мониторинг:**
- Tier 1: Latency (real-time)
- Tier 2: Precision/Recall (hourly)
- Tier 3: Feature drift (daily)
- Tier 4: Business metrics (weekly)

---

#### 2. **Component Diagram**
**Файл:** `c4_component_fraud_detection.puml`

Детальная архитектура компонентов внутри ML Service:

**Rules Engine Container:**
- Rule Evaluator (30 правил)
- Feature Lookup (из cache)
- Rules Registry (PostgreSQL)
- Feature Cache (Redis, TTL=60s)

**ML Service Container:**
- Feature Transformer
- Model Inference (LightGBM)
- Prediction Cache (in-memory)
- Cold-Start Handler (день 1-30 = rules-only)
- Model Store (v1.0 Blue, v2.0 Green для shadow mode)

**Decision Dispatcher:**
- Score Normalizer (объединить Rules + ML)
- Risk Classification (Green/Yellow/Red/Urgent)
- Decision Router (куда идет решение)
- Output Formatter (JSON Contract)
- Fallback Manager (если ML timeout)

**Quality & Monitoring:**
- Contract Validator (проверка schema)
- Latency Monitor (p50/p95/p99)
- Precision/Recall Monitor (moving window 1000)
- Feature Drift Detector (KS test)

**Data Layer:**
- Operations DB (480K/месяц)
- Customer History
- Counterparty Risk
- Decision Log

**Error Handling:**
- Inference timeout (>200ms) → Fallback
- Feature error → Fallback
- Rule error → Fallback
- Conservative default (если все упало)

---

### Для остальных инициатив (#4, #8)

#### 3. **Container Diagram: Delinquency (#4)**
**Файл:** `c4_container_delinquency.puml`

- Loan Management System → Delinquency Model (XGBoost)
- Risk Scoring Engine
- Collection Strategy Dispatcher
- Early Warning Notifications (SMS/Email)
- Collection Team Workflow (CRM)
- Payment History (Data Warehouse)

**Risk Levels:**
- Green: <10% delinquency risk → Monitor only
- Yellow: 10-30% risk → SMS reminder
- Red: >30% risk → Call collection team

---

#### 4. **Container Diagram: Churn (#8)**
**Файл:** `c4_container_churn.puml`

- CRM System → Churn Model (Random Forest)
- Retention Scoring
- Campaign Dispatcher
- Campaign Manager (retention offers)
- A/B Testing Framework
- Behavior Analytics (Data Lake)

**Risk Levels:**
- Green: <5% churn risk → No action
- Yellow: 5-20% risk → Email offer
- Red: >20% risk → Phone call + offer

---

## 🔄 Dataflow (Fraud Detection)

```
Operation arrives
  ├─ Rules Engine (2ms) → Score A
  ├─ Feature Enrichment (20ms) → Features
  ├─ ML Service (5ms) → Score B (LightGBM)
  ├─ Dispatcher (2ms) → Decision
  │  ├─ Green (<0.3): AUTO ALLOW (70%)
  │  ├─ Yellow (0.3-0.7): EXPERT REVIEW (20%)
  │  ├─ Red (≥0.7): AUTO BLOCK (8%)
  │  └─ Urgent (>500K): ESCALATE (2%)
  ├─ Execution (10ms)
  │  ├─ Store in DB
  │  ├─ Log to Audit Trail
  │  └─ Send to Expert if needed
  └─ Response to Banking API

Total latency: 47ms (SLA <100ms) ✅
```

---

## 🎭 Shadow Mode Deployment

```
Production:
  v1.0 (95% traffic) → Real decisions → DB + Response
  v2.0 (5% traffic) → Shadow results → Comparison table
  
Validation:
  ├─ Совпадение >= 95% → GO (switch to v2.0)
  ├─ Совпадение 90-95% → Разбор расхождений
  └─ Совпадение < 90% → Откатить на v1.0

Rollback: < 5 минут (UPDATE model_version в конфиге)
```

---

## 📈 Как использовать диаграммы?

**PlantUML Rendering:**
1. Online: https://www.plantuml.com/plantuml/uml/
2. VSCode Extension: `PlantUML` extension
3. Command line: `plantuml *.puml -o ./output/`

**Результат:**
- `.svg` или `.png` для документации
- Editable `.puml` исходники в этой папке

---

## ✅ Покрытие требований

| Требование | Диаграмма | Статус |
|-----------|-----------|--------|
| Container diagram | c4_container_fraud_detection.puml | ✅ |
| Component diagram | c4_component_fraud_detection.puml | ✅ |
| Выходной контракт | Component описание | ✅ |
| Fallback стратегия | Component: Fallback Manager | ✅ |
| Мониторинг | Component: Quality & Monitoring | ✅ |
| Угрозы на границах | Note sections | ✅ |
| Container для #4 | c4_container_delinquency.puml | ✅ |
| Container для #8 | c4_container_churn.puml | ✅ |

---

**See also:** 
- ai_service_architecture.md (архитектурные решения)
- component_design.md (API контракты)
- monitoring_plan.md (мониторинг по слоям)
