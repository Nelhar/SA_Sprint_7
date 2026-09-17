# Стратегия fallback: Восстановление после отказов

**Инициатива:** #1 Выявление подозрительных операций  
**SLA:** 99.9% availability (8.6 часов downtime/месяц maximum)

---

## Сценарий 1: ML Service упал

```
Operation arrives
  ├─ Try ML service (timeout 200ms)
  │   ├─ Success → Use ML score
  │   └─ Timeout/Error → Fallback
  │
  └─ Fallback to RULES ENGINE
      ├─ If Rules can decide → Use Rules score
      └─ If Rules can't decide → QUEUE TO MANUAL_REVIEW
```

**Время восстановления:** < 200ms (timeout)  
**Операции в буфере:** Если ML упал > 5 минут, ручная очередь на 30K операций

---

## Сценарий 2: Database недоступна

```
Feature query → Postgres DB (down)
  │
  └─ Use last known features (cached or default)
      ├─ Cache hit (Redis 60s TTL) → Use cached
      └─ Cache miss → Use default features (mean values)
```

**Риск:** Модель работает на дефолтных признаках (low confidence)  
**Действие:** Отправить в EXPERT_REVIEW вместо AUTO_ALLOW

---

## Сценарий 3: Cache (Redis) упал

```
Cache miss → Query DB directly (20ms)
  ├─ Success → Continue, no impact
  └─ Timeout → Use cached value from previous hour
```

**Impact:** Latency +20ms (still < 100ms SLA)

---

## Сценарий 4: Все компоненты упали (Full outage)

```
Conservative strategy:
├─ Operations < 30K AND amount < weekly_avg → AUTO ALLOW (liberal)
├─ Operations > 500K OR new_account → AUTO BLOCK (conservative)
└─ Rest → QUEUE TO MANUAL_REVIEW (wait for human)
```

**Expected duration:** < 30 минут (на восстановление)

---

## Circuit Breaker Pattern

```python
def ml_service_with_circuit_breaker(features):
    if circuit_breaker.is_open():
        return fallback_to_rules(features)
    
    try:
        score = ml_service(features)
        circuit_breaker.record_success()
        return score
    except Exception as e:
        circuit_breaker.record_failure()
        if circuit_breaker.failure_count > 5:
            circuit_breaker.open()  # Stop calling for 30s
        return fallback_to_rules(features)
```

**Conditions to open circuit:**
- 5+ consecutive failures within 60s
- Latency > 500ms (instead of 5ms)
- Error rate > 10% over 5min window

**Auto-recover:** Re-attempt after 30s timeout

---

## Queue Management (Manual Review)

```
EXPERT_REVIEW operations (score 0.3-0.7)
├─ Queued in DB table: "aml_review_queue"
├─ Priority: By amount (DESC)
├─ Batch size: 100 operations per AML expert
└─ SLA: Reviewed within 24 hours

MANUAL_REVIEW operations (ML service error)
├─ Queued separately: "manual_review_queue"
├─ Priority: By timestamp (FIFO)
├─ Alert: If queue > 1000 operations
└─ SLA: Reviewed within 4 hours (urgent)
```

---

## Replication & Backup

```
┌──────────────────┐
│ Primary Postgres │ (active, R/W)
└────────┬─────────┘
         │ replication (synchronous)
         │
┌────────▼─────────┐
│ Standby Postgres │ (hot-standby, can promote)
└──────────────────┘

Backup:
└─ Daily snapshots to S3 (retention: 30 days)
└─ Point-in-time recovery: <1 hour
```

---

## Model Rollback

```
Current: v1.0 (Precision 71%, Recall 84%)
Canary:  v2.0 (Precision 72%, Recall 85%) — 5% traffic

If v2.0 metrics degrade:
  ├─ Alert triggered
  ├─ Admin confirms rollback
  └─ Route 100% traffic back to v1.0 (< 1 min)

Kept for 30 days for forensics analysis
```

---

## Communication Plan

| Scenario | Alert | Escalation | Notify |
|----------|-------|------------|--------|
| Inference latency > 200ms | 🟡 Yellow | If > 5 min | Slack #ops |
| Service unavailable | 🔴 Red | Immediate | Page oncall + Slack |
| Manual review queue > 1000 | 🟡 Yellow | Within 30 min | Slack #aml |
| Model precision < 60% | 🔴 Red | Immediate | Page oncall + Slack |

---

**Результат:** Service восстанавливается в < 30 минут даже в worst-case scenario.

**Next:** Monitoring Plan (как мы это все мониторим).
