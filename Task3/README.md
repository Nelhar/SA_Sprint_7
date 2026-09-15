# Task 3: Архитектура AI-сервиса

---

## Что мы спроектировали

### 🏗️ Гибридная архитектура (Hybrid Automation)

```
Правила (2ms)     ML модель (5ms)      Эксперты (manual)
     ↓                  ↓                     ↓
  [Deterministic]  [Probabilistic]     [Manual review]
     ↓                  ↓                     ↓
  AUTO ALLOW (70%)  EXPERT (25%)      AUTO BLOCK (5%)
```

**Преимущество:** 70% операций обрабатываются без экспертов, 25% требуют решение эксперта, 5% автоблокируются.

---

### 🔄 Синхронный конвейер (Happy Path)

```
1. RECEIVE (10ms)      → Operation arrives
2. ENRICH (20ms)       → Add customer history + features
3. SCORE (5ms)         → LightGBM inference (from Task 2)
4. DECIDE (2ms)        → Dispatcher determines action
5. EXECUTE (10ms)      → Commit to DB + Send response

Total: 47ms (SLA <100ms) ✅
```

**Компоненты:**
1. **Rules Engine** (30 правил, <2ms) — фильтрация явных случаев
2. **ML Service** (LightGBM, 5ms) — скор fraud probability
3. **Dispatcher** (2ms) — принимает решение на основе скора
4. **Cache Layer** (Redis, 60s TTL) — ускорение feature lookup
5. **Audit Log** (Postgres) — полная история для compliance

---

### ❄️ Холодный старт (New customers)

| Возраст | Стратегия |
|---------|-----------|
| День 1-30 | RULES ONLY (нет ML, консервативно) |
| День 31-90 | ML Score + Expert override |
| День 90+ | Full ML (доверяем скору) |

**Результат:** Безопасный onboarding новых клиентов без ML-риска.

---

### 🔙 Fallback стратегия (Disaster Recovery)

**Если ML сервис упал:**
```
Try ML service (timeout 200ms)
├─ Success → Use score
├─ Timeout → Fallback to RULES ENGINE
└─ Error → Queue to MANUAL REVIEW
```

**Если все упало (worst case):**
```
Amount < 30K → AUTO ALLOW (liberal)
Amount > 500K → AUTO BLOCK (conservative)
Rest → QUEUE TO MANUAL REVIEW (wait for human)
```

**Вывод:** Service восстанавливается за < 30 минут.

---

### 🎯 Матрица автоматизации: Кто решает в каких случаях

**Проблема без матрицы:** Нет четкого разделения между "система одобряет автоматически" vs "эксперт решает" vs "система блокирует" vs "руководитель подписывает".

**Решение:** Классификация операций по РИСКУ → ДЕЙСТВИЕ:

| **Класс** | **Критерии** | **Действие** | **Контроль** | **Ответственный** | **% потока** |
|-----------|---|---|---|---|---|
| **🟢 Green (Auto-allow)** | Score < 0.3 + Amount < 30K + Whitelist контрагент | Автоматически одобрить | 5% audit/месяц | AML Lead | **70%** |
| **🟡 Yellow (Expert-review)** | Score 0.3-0.7 ИЛИ дубль за день ИЛИ Amount 30-500K | Queue to expert | 100% подтверждение | AML Expert | **20%** |
| **🔴 Red (Auto-block)** | Score ≥ 0.7 ИЛИ OFAC черный список ИЛИ Amount > 1M | Автоматически заблокировать | Log + 10% audit | AML Lead | **8%** |
| **🚨 Urgent (Escalate)** | Amount > 500K ИЛИ новый контрагент ИЛИ редкая услуга | На подпись Head | Всегда подпись + письмо | Head of AML | **2%** |

**Примеры:**
- **Green:** Микро-платеж 5K ₽ знакомому поставщику → Auto-allow за 5ms
- **Yellow:** Платеж 100K ₽ в новую страну → Expert посмотрит (5 мин)
- **Red:** Вывод 2M ₽ на черный список → Auto-block, письмо клиенту
- **Urgent:** Переводе 1M ₽ от новой ООО → Руководитель подписывает

**Ответственность:**

| Сценарий | Исход | Решение | Кто несет ответственность |
|----------|-------|---------|---|
| Green пропустил fraud | Fraud произойдет | Откатить на rules-only | AML Lead (metric monitoring) |
| Yellow: Expert ошибся | Fraud произойдет | Разбор эксперта | Head of AML (переподготовка) |
| Red заблокировал legit | Клиент жалуется | Head of AML решает | Head of AML (reputation risk) |
| Urgent: Руководитель одобрил fraud | Регулятор взял штраф | Chief Risk Officer | CRO (regulatory risk) |

**Критерий успеха пилота:**

✅ **Auto-allow rate:** 60-70% (вместо 40% ручной обработки сегодня)  
⚠️ **Если < 60%:** модель недостаточно уверена → нужна калибровка  
❌ **Если > 80%:** слишком много доверия → консервативнее  

---

### 📊 Мониторинг в Production

**Tier 1 (Real-time):**
- Inference latency p95 > 150ms → 🔴 Page oncall
- Service availability < 99.5% → 🔴 Page oncall
- Error rate > 5% → 🔴 Alert

**Tier 2 (Hourly):**
- Precision < 60% (moving window 1000) → 🔴 STOP, пересчет
- Recall < 70% → 🔴 STOP, пересчет
- ROC-AUC < 0.85 → 🔴 STOP

**Tier 3 (Daily):**
- Feature distribution drift (KS > 0.15) → Investigate
- Manual review queue > 1000 → Escalate

**Инструменты:** Prometheus + Grafana + PagerDuty

---

### 🚀 Версионирование модели

**Blue-Green deployment:**

```
v1.0 (Production)    95% трафика
└─ Precision 71%, Recall 84%

v2.0 (Canary)        5% трафика
└─ Precision 72%, Recall 85%

Откат: <1 минута, если v2.0 metrics падают
```

---

## Входные данные из Task 2

| Метрика | Значение | Откуда |
|---------|----------|--------|
| **Готовность данных** | Conditional GO (4-6 нед) | data_readiness.md |
| **Train/Val/Test** | 320K/80K/80K с 0.2% fraud | dimensioning.md |
| **Модель** | LightGBM, F1=0.77 | model_selection.md |
| **Precision** | 71% (на VAL) | model_selection.md |
| **Recall** | 84% (на VAL) | model_selection.md |
| **Inference time** | 5ms | model_selection.md |

---

## Архитектурные решения (ADR)

**Выбранные решения:**

1. ✅ **Гибридный подход** (Rules + ML) вместо pure ML
   - **Почему:** 70% автоматизации, 25% экспертное решение = баланс
   - **Риск:** Нужно поддерживать правила
   
2. ✅ **Синхронный API** (< 100ms) вместо асинхронного
   - **Почему:** Банк требует instant feedback на операцию
   - **Риск:** Требует кэширование и оптимизацию
   
3. ✅ **Dedicated Microservice** (отдельный сервис)
   - **Почему:** Независимая разработка, A/B тестирование, модели
   - **Риск:** Сетевая latency (компенсирована кэшем)
   
4. ✅ **Blue-Green deployment** (Canary для новых версий)
   - **Почему:** Быстрый откат, если новая версия плохая
   - **Риск:** Требует мониторинг 24/7
   
5. ✅ **Холодный старт 30 дней правилами**
   - **Почему:** Новые клиенты не имеют истории
   - **Риск:** Качество снижается, но безопасность выше

---

## Рекомендация для LEADERSHIP

| Вопрос | Ответ | Риск |
|--------|--------|------|
| Будет ли работать? | Да, 99.9% availability | Low |
| Быстро ли? | Да, <100ms p95 latency | Low |
| Дешево ли? | Да, 100K ₽/месяц vs 2M ₽/месяц (ручной) | Low |
| Объяснимо ли? | Да, SHAP values + Rules | Low |
| Когда запустим? | Через 2-3 недели (после Task 4) | Medium |

---

## Файлы в этой папке

| Файл | Назначение | Статус |
|------|-----------|--------|
| **ai_service_architecture.md** | Высокоуровневая архитектура, гибридный подход | ✅ |
| **component_design.md** | API контракты, database schema, cache | ✅ |
| **fallback_strategy.md** | Disaster recovery, circuit breaker, rollback | ✅ |
| **monitoring_plan.md** | SLA, alerts, dashboards, runbooks | ✅ |
| **adr_architecture.md** | Архитектурные решения с обоснованием | ✅ |
| **diagrams/README.md** | Ссылки на диаграммы C4, dataflow | ✅ |

---

## Что дальше? (Task 4)

### Вопросы, которые решает Task 4:

1. **Стоимость:** Сколько будет стоить инфраструктура?
2. **Пилот:** Как проверим на реальных данных перед продакшеном?
3. **ROI:** Какой возврат на инвестиции (экономия на экспертах)?
4. **Timeline:** Когда запустим в production?

### Входные данные для Task 4:

- ✅ Архитектура определена (Task 3)
- ✅ SLA определены (<100ms, 99.9% availability)
- ✅ Fallback стратегия (rules engine как fallback)
- ✅ Мониторинг (Prometheus + Grafana)

### Параллельная работа:

Пока дизайним Task 4 для #1, параллельно:
- Task 3 для инициативы #4 (прогноз просрочки)
- Task 3 для инициативы #8 (прогноз оттока)

---

## Ключевые числа для LEADERSHIP

| Метрика | Значение | Комментарий |
|---------|----------|-----------|
| **SLA latency** | <100ms p95 | Синхронный API, банк требует |
| **Availability** | 99.9% | 8.6 часов downtime/месяц максимум |
| **Automation rate** | 70% | Без экспертов обрабатываются 70% операций |
| **Expert capacity** | 25% | Эксперты обрабатывают только hard cases |
| **Model versions** | Canary 5% | Новые версии тестируются на 5% трафика |
| **Fallback strategy** | Rules-only | Если ML упадет, система работает на правилах |
| **Cost (infrastructure)** | ~100K ₽/мес | TBD в Task 4 |
| **Cost (manual, baseline)** | 2M ₽/мес | 25 экспертов × 80K ₽/месяц |
| **Savings** | ~1.9M ₽/мес | Потенциальная экономия |

---

**Готово к переходу на Task 4: Cost & Pilot Planning.**
