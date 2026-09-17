# План мониторинга: Метрики production и оповещения

**Инициатива:** #1 Выявление подозрительных операций  
**Stack:** Prometheus + Grafana + PagerDuty

---

## Структура мониторинга по слоям (Layered Monitoring)

```
┌─────────────────────────────────────────────────────────────┐
│              PRODUCTION FRAUD DETECTION SERVICE             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LAYER 1: Input data (Operations arriving)                 │
│  ├─ Volume check: 480K ± 10% operations/месяц             │
│  ├─ Schema validation: all required fields present          │
│  └─ Distribution: amount, country, industry not drifted    │
│           ↓                                                 │
│  LAYER 2: Rules Engine (Deterministic)                     │
│  ├─ Coverage: X% operations matched by rules               │
│  ├─ Rule execution: <2ms latency                           │
│  └─ Rule output distribution: Y% auto-allow, Z% skip ML    │
│           ↓                                                 │
│  LAYER 3: ML Model (Probabilistic scoring)                 │
│  ├─ Inference latency: <5ms p95                            │
│  ├─ Precision/Recall: > 70% / > 80%                        │
│  ├─ Feature distribution: drift < 0.10 (KS)               │
│  └─ Model version: tracking which version deployed         │
│           ↓                                                 │
│  LAYER 4: Decision Dispatcher (Logic)                      │
│  ├─ Classification accuracy: score → decision              │
│  ├─ Throughput: >1000 ops/sec                             │
│  └─ SLA compliance: p95 < 100ms total                       │
│           ↓                                                 │
│  LAYER 5: Expert Review (Human-in-loop)                    │
│  ├─ Queue depth: < 500 pending reviews                     │
│  ├─ Expert accuracy: correction_rate 10-20%               │
│  ├─ Quality control: post_error_rate < 3%                 │
│  └─ Workload: avg 3-5 minutes per decision                │
│           ↓                                                 │
│  LAYER 6: Business Outcome (Impact)                        │
│  ├─ Fraud caught: target 84% recall                        │
│  ├─ Auto-allow rate: 70% ± 5%                             │
│  ├─ Cost per decision: trend downward                      │
│  └─ Expert satisfaction: survey feedback                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Алерты по слоям (Cascading alerts)

| Слой | Red Alert | Yellow Alert | Ответственный | Время ответа |
|------|-----------|--------------|---|---|
| **LAYER 1: Input** | Volume < 50% baseline | Volume < 90% | Data Team | 30 мин |
| **LAYER 2: Rules** | Coverage < 50% | Coverage < 70% | Rules Owner | 30 мин |
| **LAYER 3: ML** | Precision < 60% | Precision < 70% | ML Engineer | 15 мин |
| **LAYER 4: Dispatcher** | Latency > 200ms | Latency > 150ms | DevOps | 5 мин |
| **LAYER 5: Expert** | Queue > 2000 | Queue > 1000 | AML Lead | 30 мин |
| **LAYER 6: Business** | Recall < 70% | Recall < 80% | CRO | 1 час |

### Когда что проверять

```
Every 10 seconds (Real-time):
├─ LAYER 4: Dispatcher latency p95
└─ LAYER 1: Input volume rate (ops/min)

Every 1 minute:
├─ LAYER 1: Schema validation errors
├─ LAYER 2: Rules coverage
└─ LAYER 4: Service availability

Every 1 hour:
├─ LAYER 3: Model precision/recall (moving 1000)
├─ LAYER 3: ROC-AUC on moving window
└─ LAYER 5: Queue depth, correction rate

Every 1 day:
├─ LAYER 1: Feature distribution drift (KS)
├─ LAYER 3: Data drift detection
├─ LAYER 5: Expert per-person statistics
└─ LAYER 6: Business metrics aggregation

Every 1 week:
├─ LAYER 5: Cohen's kappa (expert agreement)
├─ All layers: Trend analysis
└─ LAYER 6: ROI recalculation
```

---

## Tier 1: Service Health (Real-time)

| Метрика | Red Line | Yellow Line | Check | Tool |
|---------|----------|-------------|-------|------|
| **Inference latency p95** | > 150ms | > 100ms | Every 10s | Prometheus |
| **Service availability** | < 99.5% | < 99.9% | Every 1min | Prometheus |
| **Error rate** | > 5% | > 2% | Every 1min | Prometheus |
| **Circuit breaker open** | Any | N/A | Real-time | Prometheus |

**Alert action:**
- 🔴 Red → Page oncall immediately
- 🟡 Yellow → Slack notification

---

## Tier 2: Model Quality (Hourly)

| Метрика | Red Line | Yellow Line | Window | Tool |
|---------|----------|-------------|--------|------|
| **Precision** | < 60% | < 70% | 1000 samples | Custom |
| **Recall** | < 70% | < 80% | 1000 samples | Custom |
| **ROC-AUC** | < 0.85 | < 0.88 | 1000 samples | Custom |
| **Model version** | Outdated | > 6 months | Deployment log | Custom |

**Implementation:**

```python
def compute_metrics_hourly():
    # Fetch last 1000 predictions + expert decisions
    predictions = get_predictions_since(now() - 1hour)
    
    precision = len(tp) / len(tp + fp)
    recall = len(tp) / len(tp + fn)
    auc = roc_auc_score(y_true, y_pred)
    
    prometheus_gauge("model_precision", precision)
    prometheus_gauge("model_recall", recall)
    prometheus_gauge("model_auc", auc)
```

---

## Tier 3: Data Quality (Daily)

| Метрика | Expected | Threshold | Action |
|---------|----------|-----------|--------|
| **Feature distribution drift (KS)** | KS < 0.10 | > 0.15 | Investigate |
| **Missing features** | 0% | > 0.1% | Alert |
| **Null values in key fields** | 0% | > 1% | Alert |
| **Manual review queue size** | < 500 | > 1000 | Escalate |

---

## Tier 4: Business Metrics (Weekly)

| Метрика | Baseline | Target |
|---------|----------|--------|
| **Auto-allow rate** | 40% | 35-45% |
| **Expert review rate** | 50% | 45-55% |
| **Auto-block rate** | 10% | 5-15% |
| **Expert review time (avg)** | 2 min | < 5 min |
| **Cost per decision** | 2 ₽ | < 0.2 ₽ |

---

## Dashboard (Grafana)

```
┌────────────────────────────────────────────────────┐
│             FRAUD DETECTION SERVICE                │
├────────────────────────────────────────────────────┤
│                                                    │
│  [Inference Latency p95]  [Service Availability]   │
│       45ms (GREEN)              99.95% (GREEN)     │
│                                                    │
│  [Precision (1h moving)]  [Recall (1h moving)]     │
│      71% (YELLOW)            84% (GREEN)           │
│                                                    │
│  [Error Rate]  [Circuit Breaker Status]            │
│      0.5%      [Closed - OK]                       │
│                                                    │
│  [Manual Review Queue]  [Model Version]            │
│      347 pending        v1.0 (Oct 2024)            │
│                                                    │
│  [Request Rate]         [Auto-allow vs Block]      │
│      25K ops/min        40% / 10% / 50%            │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Alert Rules (Prometheus)

```yaml
groups:
  - name: fraud_detection
    rules:
      - alert: HighLatency
        expr: histogram_quantile(0.95, inference_latency_ms) > 150
        for: 5m
        annotations:
          severity: critical
          action: Page oncall
      
      - alert: LowPrecision
        expr: model_precision < 0.60
        for: 1h
        annotations:
          severity: critical
          action: Page oncall + freeze deployments
      
      - alert: LowRecall
        expr: model_recall < 0.70
        for: 1h
        annotations:
          severity: critical
          action: Page oncall + notify Compliance
      
      - alert: ServiceDown
        expr: up{service="fraud_detection"} == 0
        for: 1m
        annotations:
          severity: critical
          action: Page oncall immediately
```

---

## Runbook (Oncall guide)

**Alert: HighLatency**
```
1. Check ML service status: curl localhost:8000/health
2. Check database connection: psql -c "SELECT 1"
3. Check Redis status: redis-cli ping
4. If all OK → check network latency to Postgres
5. If latency still high → rollback last deployment
6. If issue persists → scale up ML service replicas
```

**Alert: LowPrecision**
```
1. Fetch recent expert decisions: SELECT * FROM aml_review_queue LIMIT 100
2. Check if expert behavior changed (new expert? new rules?)
3. Run diagnostic on recent false positives (model_score < 0.5, expert_decision = BLOCK)
4. Check feature distribution drift
5. If feature drift > 0.15 (KS) → trigger model retraining
6. If behavior changed → consult with AML team lead
```

---

## Tier 5: Expert Behavior (Quality Control)

**Почему это важно:** Эксперт может согласиться с системой, но это не значит что он проверил результат тщательно. Две метрики показывают правду:

| Метрика | Что означает | Норма | Красный флаг | Action |
|---------|---|---|---|---|
| **Correction rate** | % операций где эксперт изменил решение ML | 10-20% | > 25% | Модель плохо работает |
| **Post-confirmation error** | % ошибок найденных ПОСЛЕ подтверждения | 1-3% | > 5% | Эксперт не проверяет (formality) |

### Как читать эти метрики вместе

```
Scenario A: Correction ↑, Post-error ↓
├─ Эксперт проверяет тщательно
├─ Находит ошибки системы ДО подтверждения ✅ GOOD
└─ Система работает OK, эксперты alert и careful

Scenario B: Correction ↓, Post-error ↑ ❌ CRITICAL
├─ Эксперт не проверяет (formality, rubber stamp)
├─ Ошибки обнаруживают клиенты/аудиторы ПОСЛЕ
└─ Требуется переподготовка эксперта IMMEDIATELY

Scenario C: Correction ↑, Post-error ↑ ❌ CRITICAL
├─ Эксперт очень active (часто меняет)
├─ НО ошибки все равно проходят через него
└─ Модель деградировала (data drift) ИЛИ эксперт растерялся

Scenario D: Correction ↓, Post-error ↓ ✅ IDEAL
├─ Все работает хорошо
├─ Система и эксперты в синхронизации
└─ Этап "mature system"
```

### Как измерять

**Correction rate:**
```
= COUNT(decisions_where_expert_changed_system_output) 
  / COUNT(all_expert_reviews) per week
```

**Post-confirmation error:**
```
= COUNT(confirmed_operations_found_wrong_later) 
  / COUNT(confirmed_operations) per week

Где "found wrong later":
- Клиент подал жалобу (complain_date > decision_date)
- Аудитор обнаружил при проверке  
- Регулятор указал при ревью
```

### Таблица мониторинга Expert Behavior

| Метрика | Частота | Red Line | Yellow Line | Действие |
|---------|---------|----------|------------|---------|
| **Correction rate** | Daily | > 25% | > 18% | Разбор: почему эксперт часто меняет? |
| **Post-error rate** | Daily | > 5% | > 3% | Проверка квалификации эксперта |
| **Cohen's kappa** (согласованность) | Weekly | < 0.70 | < 0.75 | Уточнить гайдлайны разметки |
| **Avg time per case** | Daily | > 5 min | > 3 min | Проверить UX интерфейса |

### Validation Set (Контрольный набор)

**50 операций в неделю с заранее известным правильным ответом:**

```
Процесс:
1. Разметили независимо (не система, не основной эксперт)
2. Пропускаем через систему + даем эксперту
3. Сравниваем результат с истиной

Пример:
Truth:     fraud (эксперт A + эксперт B согласились)
System:    0.72 (expert review needed)
Expert:    подтвердил fraud ✅ CORRECT

Если эксперт меняет много случаев из контроля:
→ Системе нужна переобработка ИЛИ нужна переподготовка эксперта
```

### Escalation Alert

```python
def check_expert_behavior(expert_id, window="1week"):
    corrections = COUNT(changed decisions)
    total = COUNT(all decisions)
    correction_rate = corrections / total
    
    if correction_rate > 0.25:
        alert("EXPERT_BEHAVIOR_ALERT", {
            "expert": expert_id,
            "correction_rate": correction_rate,
            "message": "Эксперт часто меняет решение ML. Система может быть плохая или эксперт может не доверять ML.",
            "action": "Разбор с AML Lead"
        })
```

---

## Logging (Structured)

```json
{
  "timestamp": "2024-10-16T10:30:45.123Z",
  "operation_id": 12345,
  "account_id": 67890,
  "event": "fraud_score_computed",
  "rules_score": null,
  "ml_score": 0.62,
  "decision": "EXPERT_REVIEW",
  "inference_time_ms": 5,
  "model_version": "v1.0",
  "top_factors": [
    {"name": "foreign_country", "impact": 0.32},
    {"name": "amount_deviation", "impact": 0.24}
  ]
}
```

**Log aggregation:** ELK Stack (Elasticsearch + Kibana)  
**Retention:** 30 days hot, 1 year cold (S3)

---

## On-call Rotation

```
Primary: ML-Engineer (24/7 rotation, 1 week)
Secondary: Architect (escalation after 15 min)
Tertiary: ML Team Lead (escalation after 30 min)

Severity:
- SEV1 (Service down): 5 min response
- SEV2 (Metrics degraded): 15 min response
- SEV3 (Warnings): Next business day
```

---

## Review Cadence

| Frequency | Reviewers | Items |
|-----------|-----------|-------|
| **Weekly** | ML-Engineer + Oncall | Alerts, False positives, Latency trends |
| **Monthly** | Architect + PM | ROI, Cost per decision, Expert feedback |
| **Quarterly** | Head of ML + AML Lead | Model drift, Retraining needs, Roadmap |

---

**Next:** ADR Architecture (формальное решение).
