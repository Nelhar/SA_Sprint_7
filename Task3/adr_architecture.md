# ADR: Архитектура AI-сервиса

**Статус:** ✅ ПРИНЯТО  
**Дата:** 2024-10-16  
**Архитектор:** Solution Architect (ML)  

---

## Проблема

Как архитектировать AI-сервис для выявления подозрительных операций, если:

1. **Производительность требуется:** <100ms на p95 (банк требует синхронный API)
2. **Риск отказа:** Модель должна быть объяснима (SHAP), иначе регулятор запретит
3. **Масштабируемость:** 25M операций/месяц = 270 операций/сек в пиках
4. **Отказоустойчивость:** 99.9% availability (8.6 часов downtime в месяц допустимо)
5. **Холодный старт:** Новые клиенты не имеют истории → нужна гибридная логика

---

## Решение: Hybrid Automation Pattern

### Компоненты

```
Правила (Deterministic)     ML модель (Probabilistic)    Эксперты (Manual)
├─ 30 правил               ├─ LightGBM (F1=0.77)         ├─ 25 AML-экспертов
├─ Вычисляются за 2ms      ├─ Inference за 5ms           ├─ Решают за 2 мин
└─ 100% точность          └─ Precision 71%, Recall 84%  └─ Субъективно

        ↓                         ↓                            ↓
        └──────────────┬──────────┬───────────────────────────┘
                       │          │
                   [Dispatcher]───┤
                       │          │
          ┌────────────┴──────────┴───────────┐
          │                                   │
    Score < 0.3 (70%)               0.3-0.7 (25%)           Score ≥ 0.7 (5%)
    AUTO ALLOW                   EXPERT REVIEW             AUTO BLOCK
    (нет экспертов)              (экспертное решение)      (высокий риск)
```

### Преимущества

| Решение | Преимущество | Компромисс |
|---------|-------------|-----------|
| **Правила + ML** | 70% полностью автоматизировано | Нужно поддерживать список правил |
| **Гибридный подход** | Эксперты только для hard cases (25%) | Требуется обучение экспертов на ML скор |
| **Fallback to Rules** | Если ML упадет, система продолжит работать | Качество снижается на ~10% |
| **Холодный старт (новые клиенты)** | Безопасный консервативный подход | Первые 30 дней без ML |

---

## Архитектурные решения

### 1. Синхронный vs Асинхронный API

**Выбор:** ✅ **Синхронный** (операция ждет ответа)

| Аспект | Синхронный | Асинхронный | Выбор |
|--------|-----------|-------------|-------|
| Latency | <100ms | 1-5 сек | Sync |
| User experience | Instant feedback | Delayed | Sync |
| Complexity | Выше (cache, timeout) | Ниже | Sync (worth it) |
| Scalability | Требует горизонтальное масштабирование | Легче | Sync (OK) |
| Cost | Higher (compute) | Lower | Sync (business value) |

**Почему:** Банк требует instant ответ на операцию. Асинхронный был бы дешевле, но плохой UX.

---

### 2. Где живет модель?

**Выбор:** ✅ **Dedicated Microservice** (не в мейн-приложении)

| Место | Плюсы | Минусы | Выбор |
|-------|-------|--------|-------|
| Embedded (в мейн-сервис) | Простая интеграция, низкая latency | Hard to update, SPOF | ❌ |
| Microservice | Независимая разработка, A/B тестирование | Сетевая latency | ✅ |
| Serverless (AWS Lambda) | Cheap, autoscale | Cold start (500ms), latency > SLA | ❌ |
| Edge (в базе) | Минимальная latency | Сложная синхронизация моделей | ❌ |

**Архитектура:**

```
API Gateway (Kong)
    ↓
Load Balancer (Nginx)
    ↓
┌─────────────────────┐
│ ML Service Pool     │
│ (3x replicas)       │
│ FastAPI + Python    │
│ <100ms latency SLA  │
└─────────────────────┘
    ↓
Shared resources:
├─ LightGBM model (2MB, shared)
├─ Redis cache (features)
└─ Postgres (operations log)
```

---

### 3. Model versioning & Rollback

**Выбор:** ✅ **Blue-Green deployment с Canary**

```
v1.0 (Production)       95% трафика
└─ Precision 71%
└─ Recall 84%
└─ Deployed Oct 2024

v2.0 (Canary)           5% трафика
└─ Precision 72%
└─ Recall 85%
└─ Deployed Dec 2024 (тест)
```

**Откат:** Если v2.0 metrics падают, автоматически откат на v1.0 за < 1 минуту.

---

### 4. Холодный старт (новые клиенты)

**Выбор:** ✅ **Гибридный подход с временным окном**

| Возраст | Логика | Обоснование |
|---------|--------|-------------|
| День 1-30 | RULES ONLY | Нет истории для ML, консервативный подход |
| День 31-90 | ML Score + Expert override | Начинаем доверять ML, но эксперт может выше |
| День 90+ | Full ML | Полное доверие скору |

**Правила для дней 1-30:**

```
if amount < 50K and industry_known:
    AUTO_ALLOW
elif amount > 500K or foreign:
    AUTO_BLOCK
else:
    EXPERT_REVIEW
```

---

### 5. Cache Strategy

**Выбор:** ✅ **Redis с 60s TTL для features**

| Сценарий | Решение | Rationale |
|----------|---------|-----------|
| Feature cache hit | Use cached features (5ms) | Fast path for repeated operations |
| Feature cache miss | Query DB (20ms) | Accurate, fresh data |
| Cache layer down | Use default features | Fallback, low confidence score |

**TTL 60s:** Operations от одного аккаунта в пределах минуты → одинаковые признаки (customer_lifetime, account_age и т.д.)

---

## Допущения (Assumptions)

| # | Допущение | Статус | Риск | Fallback |
|---|-----------|--------|------|----------|
| **A1** | Latency <100ms достижима с LightGBM | ✅ Verified | Low | If not, switch to LogReg (simpler, faster) |
| **A2** | 3 replicas хватает для 270 ops/sec | 📋 TBD | Medium | Auto-scale при > 80% CPU |
| **A3** | Rules engine можно обновить за день | ✅ Assumed | Low | If not, deploy ML-only fallback |
| **A4** | Эксперты могут обработать 25% операций | ⏳ TBD | Medium | If not, switch some to auto-allow |
| **A5** | Redis будет stable (99.95%) | 📋 TBD | Low | Fallback to DB queries |
| **A6** | DPO одобрит модель (регуляторика OK) | ⏳ Pending | High | If not, deploy rules-only until approval |

---

## Риски и смягчение

| Риск | Вероятность | Влияние | Смягчение |
|------|------------|---------|-----------|
| **Inference latency > 150ms** | Medium | High (UX) | Caching, model optimization, horizontal scaling |
| **Model performance degrades in prod** | Medium | High (business) | Monitoring (hourly), quick rollback, human loop |
| **Service unavailability** | Low | Critical | Fallback to rules, circuit breaker, replication |
| **Feature leakage** | Low | High (compliance) | Code review, SHAP analysis, validation tests |
| **Regulation blocks ML** | Low | Critical | Rules-only fallback (already designed) |
| **New customer cold-start** | High | Medium | 30-day rule-based warmup (mitigated) |

---

## Trade-offs

| Trade-off | Выбор | Альтернатива | Почему |
|-----------|-------|-------------|--------|
| **Complexity vs Speed** | Microservice | Embedded | Worth the latency cost for modularity |
| **Cost vs Quality** | GPU inference | CPU | GPU too expensive for 5ms inference (can do CPU) |
| **Accuracy vs Latency** | LightGBM | Neural network | LightGBM faster, simpler, comparable accuracy |
| **Automation vs Control** | 70% auto | 100% auto | Keep 25% for experts (risk mitigation) |

---

## Зависимости и блокеры

**READY TO DEPLOY (Task 3):**
- ✅ Architecture defined
- ✅ Components designed
- ✅ Fallback strategy

**BLOCKED (from Task 4):**
- ⏳ Infrastructure sizing (AWS/GCP/on-prem?)
- ⏳ Cost estimation
- ⏳ Deployment plan (CI/CD, Terraform)

**BLOCKED (from DPO):**
- ⏳ Regulatory approval for model

---

## Утверждение

- ✅ **Архитектор:** Согласен (2024-10-16)
- ✅ **ML-Engineer:** Ready to code (2024-10-16)
- ✅ **DevOps/Platform:** Ready to deploy (2024-10-16)
- ⏳ **Compliance/DPO:** Pending approval (2024-10-20 expected)

---

## Документация

- **ai_service_architecture.md** — High-level design
- **component_design.md** — API contracts, database schema
- **fallback_strategy.md** — Disaster recovery, circuit breaker
- **monitoring_plan.md** — SLA, alerts, dashboards

---

**Next:** Task 4 (Cost & Pilot Planning).
