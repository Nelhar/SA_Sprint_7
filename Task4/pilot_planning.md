# Планирование пилота: Поэтапное развертывание

**Инициатива:** #1 Выявление подозрительных операций  
**Длительность пилота:** 8 недель  

---

## Фаза 1: Synthetic Data (Неделя 1-2)

**Цель:** Проверить все компоненты работают (dev/staging)

**Данные:** 10K synthetic операций (сгенерированы, не реальные)

**Тесты:**

| Сценарий | Ожидание | Статус |
|----------|----------|--------|
| Rules engine works | Score generated | ✅ |
| ML service responds | <100ms latency | ✅ |
| Decision dispatcher | Action decided | ✅ |
| Cache hit rate | >80% | ✅ |
| Error handling | Fallback works | ✅ |
| Monitoring alerts | Fired when > 150ms | ✅ |
| Audit log | All decisions logged | ✅ |

**Выход:** All green, готовы к реальным данным.

---

## Фаза 2: Real Data, Limited Scope (Неделя 3-6)

**Цель:** Тестировать на реальных операциях, но ограниченная аудитория

**Данные:** 50K реальных операций из последнего месяца

**Скоп:** 10% операций (микросегмент, ~2.5M операций/месяц выборка)

```
100% потока операций
   │
   ├─ 90% → Current rules engine (prod, no change)
   └─ 10% → NEW ML service (pilot, parallel run)
           ├─ ML score computed
           └─ Expert comparison (is ML better than rules?)
```

**Метрики:**

| Метрика | Baseline (Rules) | Target (ML) | Статус |
|---------|-----------------|----------|--------|
| **Precision** | 80% (rules) | 71% (ML) | ⚠️ Lower (expected) |
| **Recall** | 70% (rules) | 84% (ML) | ✅ Higher (good!) |
| **Latency** | 50ms (rules) | 47ms (ML) | ✅ Faster |
| **Errors** | 0 | <0.1% | ✅ Acceptable |

**Выход:** 
- ✅ ML recall лучше (ловим больше мошенничества)
- ⚠️ Precision чуть ниже (больше ложных алертов для экспертов)
- ✅ Общее улучшение quality (F1 = 0.77 vs 0.75 от rules)

---

## Фаза 3: Real Data, Gradual Ramp (Неделя 7-8)

**Цель:** Постепенно увеличивать трафик, мониторить

**Рамп-ап:**

```
Week 7a: 20% трафика на ML
  └─ Continue parallel validation
  └─ Expert feedback on scores

Week 7b: 50% трафика на ML
  └─ Validate latency SLA
  └─ Check error handling

Week 8a: 80% трафика на ML
  └─ Monitor scaling (CPU, memory)
  └─ Check fallback strategy works

Week 8b: 100% трафика на ML
  └─ Production deployment
  └─ Keep old rules as fallback (circuit breaker)
```

**Параллельные проверки:**

| Check | Interval | Action if fails |
|-------|----------|-------------------|
| Latency p95 < 150ms | Every 10min | Rollback to rules |
| Error rate < 5% | Every 1hour | Investigate + rollback if critical |
| Precision > 60% | Every 1hour | Alert oncall, but continue |
| Recall > 70% | Every 1hour | Alert oncall, but continue |
| Manual review queue | Every 30min | Scale up experts if > 1000 |

---

## Post-Pilot Decision Framework

### ✅ GO — Продолжить на production

**ВСЕ три условия должны быть выполнены одновременно:**

| # | Условие | Порог | Проверка | Ответственный |
|---|---------|-------|---------|---|
| **1** | Precision на реальных данных | ≥ 70% | Validation set results | ML-Engineer |
| **2** | Recall на реальных данных | ≥ 80% | Validation set results | ML-Engineer |
| **3** | Эксперты согласны | ≥ 80% positive feedback | Survey/meeting | AML Lead |

**Дополнительные зеленые флаги (желательны):**
- Latency p95 < 150ms ✅
- Error rate < 1% ✅
- Expert queue length < 1000 ✅
- Monitoring alerts working ✅
- Runbooks complete ✅
- Compliance sign-off obtained ✅

**Решение:** GO ✅ → Развернуть на production, перейти на full traffic

---

### 🛑 NO-GO — Остановить и переделать

**ЛЮБОЕ из этих условий вызывает NO-GO:**

| # | Condition | Триггер | Действие | Ответственный |
|---|-----------|---------|---------|---|
| **1** | Precision слишком низкая | < 60% | Разбор модели, переобучение | ML-Engineer |
| **2** | Recall слишком низкий | < 70% | Проверка на дрейф данных | ML-Engineer |
| **3** | Latency неприемлемо | > 200ms на p95 | Оптимизация или откат | DevOps |
| **4** | Эксперты жалуются | < 60% positive feedback | Переподготовка или доработка | AML Lead |
| **5** | Ошибки в production | > 5% error rate | Поиск причины, hotfix | ML-Engineer |
| **6** | Compliance не одобрил | DPO says "No" | Решить regulatory issues | DPO |

**Решение:** NO-GO 🛑 → 
- Остановить трафик на ML (откатить на rules-only)
- Провести root-cause analysis (1-2 недели)
- Исправить проблему (переобучение / архитектура / данные)
- Переделать пилот с фазы 2

**Типичные причины NO-GO:**
```
NO-GO 1: Precision 55% (слишком много ложных алертов)
→ Причина: Selection bias в training set
→ Решение: Retroactive разметка расширена, переобучение

NO-GO 2: Recall 60% (пропускаем реальное мошенничество)
→ Причина: Дата дрейф (новые типы fraud, которых модель не видела)
→ Решение: Добавить новые примеры, переобучить

NO-GO 3: Эксперты: "Система помогает на 40%" (ожидали 80%)
→ Причина: Модель не уверена, много score 0.4-0.6 (эксперты все равно проверяют)
→ Решение: Калибровка score → поднять threshold

NO-GO 4: Latency 300ms (система медленная, клиенты жалуются)
→ Причина: XGBoost вместо LightGBM, или недостаточно ресурсов
→ Решение: Масштабировать pods или переоптимизировать
```

---

### 🔄 PIVOT — Изменить scope

**ЕСЛИ NO-GO, но не критичный → PIVOT:**

| Option | Trigger | Decision | Impact |
|--------|---------|----------|--------|
| **Pivot 1: Reduce Scope** | Recall < 70% на всех operation types | Focus only on high-value (>100K) | Автоматизируем 40% операций вместо 100% |
| **Pivot 2: Expert Filtering** | Precision 65% (много false positives) | Run ML, then expert 2-level review | Больше work for experts, but higher quality |
| **Pivot 3: Hybrid Model** | Latency issue on GPU inference | Use LightGBM for 90%, LLM for 10% | Split by operation type |
| **Pivot 4: Delayed Go** | Compliance not ready | Continue on rules, pilot ML in shadow mode | Отложить production на 2-4 недели |

**Решение:** PIVOT 🔄 →
- Переживаем пилот с новым scope
- Expectations: 6-8 недель доп. работы
- Go/No-Go criteria остаются те же, но для уменьшенного scope

---

### Monitoring during pilot (Real-time dashboard)

**Во время пилота (недели 3-8) считаем метрики ежедневно:**

```
DAILY METRICS (08:00 UTC, для GO/NO-GO решения):
├─ Precision (% correct fraud detections)
├─ Recall (% fraud caught)
├─ F1 (harmonic mean, target ≥ 0.75)
├─ Latency p50/p95/p99
├─ Error rate (% failed requests)
├─ Expert feedback (sentiment)
└─ Queue length (pending expert review)

RED LINE triggers immediate investigation:
├─ Precision < 60% → Page oncall
├─ Recall < 70% → Page AML Lead
├─ Latency > 200ms → Page DevOps
├─ Error > 5% → Page ML-Engineer
└─ Expert feedback < 50% positive → Schedule meeting
```

---

## Rollback Plan

**If anything goes wrong:**

```
Severity 1 (Service down):
  └─ Immediate rollback to rules engine (< 1 min)
  └─ Page oncall

Severity 2 (Metrics degrade):
  └─ Wait 30 min for confirmation
  └─ If still bad → rollback to previous version

Severity 3 (Minor issues):
  └─ Continue monitoring
  └─ Schedule fix for next release
```

---

## Timeline

```
Week 1-2:   Synthetic data (dev/staging)
Week 3-6:   Real data, 10% traffic
Week 7-8:   Gradual ramp (20% → 50% → 80% → 100%)
Week 9:     Post-mortem, final go/no-go decision
Week 10+:   Full production (if go)
```

**Total:** 10 weeks from pilot start to production.

---

**Next:** ROI Calculation (how much $$ do we save?).
