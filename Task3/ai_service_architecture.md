# AI-сервис архитектура: Выявление подозрительных операций

**Инициатива:** #1 Выявление подозрительных операций  
**Входные данные из Task 2:** LightGBM, F1=0.77, Precision 71%, Recall 84%  
**SLA:** <100ms на p95, 99.9% availability  

---

## Высокоуровневая архитектура

```
┌────────────────────────────────────┐
│    БАНК МСБ (20-25M операций/мес) │
├────────────────────────────────────┤
│                                    │
│  Операция → [Rules Engine] ────────┤
│               ↓                    │
│            [ML Service:            │
│             LightGBM]              │
│               ↓                    │
│         [Decision Logic]           │
│        /      |      \             │
│    AUTO   EXPERT   AUTO            │
│   ALLOW   REVIEW   BLOCK           │
│      \      |      /               │
│       [Dispatcher]                 │
│            ↓                       │
│      Ops DB + Audit Log            │
└────────────────────────────────────┘
```

---

## Конвейер обработки (Happy Path)

```
1. RECEIVE (10ms)
   Operation → Rules (amount < 30K?) → Score A

2. ENRICH (20ms)
   Customer history + Counterparty risk → Features

3. SCORE (5ms)
   LightGBM inference → Probability fraud (0-1)

4. DECIDE (2ms)
   Score < 0.3 → AUTO ALLOW
   0.3 ≤ Score < 0.7 → EXPERT REVIEW
   Score ≥ 0.7 → AUTO BLOCK

5. EXECUTE (10ms)
   Commit to DB + Response

Total: 47ms (SLA <100ms) ✅
```

---

## Правила vs ML логика

| Правило | ML Score | Действие | Когда |
|---------|----------|----------|--------|
| Сумма < 30K + Whitelist | Skip | AUTO ALLOW | 40% операций |
| Сумма < 50K (новый клиент) | Skip | AUTO ALLOW | 10% новых |
| Черный список OFAC | Skip | AUTO BLOCK | < 1% |
| Остальное | Use | ML Score | 49% операций |

---

## Холодный старт (New customers)

| Возраст | Логика |
|---------|--------|
| День 1-30 | RULES ONLY |
| День 31-90 | ML Score + Expert override |
| День 90+ | Full ML (доверяем скору) |

---

## Эскалация по качеству vs по доступности (CRITICAL DISTINCTION)

**ВАЖНО:** Это РАЗНЫЕ процессы с разными действиями!

### Эскалация ПО КАЧЕСТВУ (Quality Escalation) 

Когда СИСТЕМА работает, но РЕЗУЛЬТАТ сомнительный:

| Причина | Пример | Действие | Куда |
|---------|--------|---------|------|
| Контракт не пройден | Schema error (сумма = null) | Log alert | expert_queue |
| Инвариант не сошелся | amount_total ≠ sum(items) | Log warning | expert_queue |
| Источник отсутствует | Поле "контрагент" пусто, но сумма OK | Log | expert_queue |
| Источники противоречат | Разные суммы в разных полях | Log alert | expert_queue |
| Score на пороге | 0.48-0.52 (не уверены) | Log info | expert_queue |

**Куда идет:** expert_queue → эксперт ПРОВЕРИТ и решит  
**Время:** Стандартное (5-10 минут на проверку)  
**Результат:** Эксперт подтверждает ИЛИ меняет решение

### Эскалация ПО ДОСТУПНОСТИ (Availability Escalation)

Когда СИСТЕМА не может дать ответ:

| Причина | Пример | Действие | Куда |
|---------|--------|---------|------|
| ML service timeout | > 200ms | Retry once | retry_queue |
| ML service error 503 | Service unavailable | Retry 3x | retry_queue |
| Database timeout | Postgres не отвечает | Retry + cache | retry_queue |
| Feature lookup failed | Redis cache miss + DB down | Use defaults | retry_queue |
| Circuit breaker open | Too many failures | Fall back | rules_engine |

**Куда идет:** retry_queue → система ПОВТОРЯЕТ, потом fallback  
**Время:** < 5 сек на retry, потом автоматический fallback  
**Результат:** Либо успех на retry, либо fallback на RULES  

### Правильная логика ветвления

```python
def process_operation(operation):
    try:
        # Try ML scoring
        ml_score = call_ml_service(operation, timeout=200ms)
        
    except TimeoutError:
        # AVAILABILITY ESCALATION
        log("ML timeout, retrying...")
        time.sleep(100ms)
        ml_score = call_ml_service(operation, timeout=200ms)
        
        if still_timeout:
            # Fallback to rules
            log("ML unavailable, using rules engine")
            return rules_engine.score(operation)
    
    except ValidationError as e:
        # QUALITY ESCALATION
        log(f"Quality issue: {e}", level=WARNING)
        return expert_queue.add(operation, reason="validation_failed")
    
    # Check score validity
    if ml_score.is_valid():
        decision = dispatcher.decide(ml_score)
        
        if decision == "NEEDS_REVIEW":
            # QUALITY ESCALATION (score on edge)
            return expert_queue.add(operation, reason="score_uncertain")
        else:
            return decision
    else:
        # QUALITY ESCALATION (invalid score)
        log(f"Invalid score: {ml_score}", level=ERROR)
        return expert_queue.add(operation, reason="score_invalid")
```

---

## Fallback (Failover) — Что делать если ВСЕ упало

```
Try ML service (timeout 200ms)
├─ Success → Use score, go to dispatcher
├─ Timeout/Error → Fall back to RULES ENGINE (100% deterministic)
│   ├─ Success → Use rules score
│   └─ Rules also fail → Conservative defaults
│       ├─ Amount < 30K → AUTO ALLOW (liberal, for UX)
│       └─ Amount > 500K → QUEUE to MANUAL (safe, for risk)
└─ All systems down → Reject operation, notify oncall (< 30 min recovery SLA)
```

---

## Версионирование модели

**Blue-Green deployment:**

```
v1.0 (Production) ← 95% трафика
v2.0 (Canary) ← 5% трафика

Откат за < 1 минуту, если метрики падают.
```

---

## Shadow Mode: Безопасный выкат версий (Zero-risk Rollout)

**Проблема:** Как перейти на новую версию модели, если она лучше? Но если она лучше только на тестовом наборе, а в production хуже?

**Решение:** Shadow Mode — новая версия обрабатывает 100% трафика **параллельно**, но результаты не идут в production.

### Архитектура Shadow Mode

```
Operation arrives
    ↓
    ├─ v1.0 (Production) → Returns REAL decision (user sees)
    │   ├─ amount ✅ approved
    │   └─ Save to DB
    │
    ├─ v2.0 (Shadow) → Compute score (user does NOT see)
    │   ├─ amount ✅ would approve (or ❌ would block)
    │   └─ Save to SHADOW table (for analysis only)
    │
    └─ Comparison engine
        ├─ Did v1.0 and v2.0 agree? (90% →ок, 70% →разбор)
        └─ On what segments v2.0 is better? (precision/recall/f1)
```

### Процедура выката

**Шаг 1: Диагностический набор (20 примеров)**

```python
# Запустить v2.0 на тех же 20 примерах из Task 2
# Сравнить результаты:

if (precision_v2_new < precision_v1_old - 5%) or (recall_v2 < recall_v1 - 5%):
    print("❌ FAIL: v2.0 хуже на диагностике")
    print("Не развертываем, ищем bug")
    sys.exit(1)
else:
    print("✅ PASS: v2.0 прошла диагностику")
```

**Шаг 2: Shadow Mode (7-14 дней параллельной работы)**

Включаем shadow mode на production → v2.0 обрабатывает 100% трафика, но решения не используются.

**Шаг 3: Сравнение результатов**

| Вопрос | Ответ | Действие |
|--------|-------|----------|
| На скольких операциях результаты расходятся? | > 10% | 🔴 Разобраться в причине, может быть, дрейф данных |
| На каких сегментах расходятся? | Если на редких | ⚠️ Возможно нужна доразметка редких примеров |
| Где v2.0 выигрывает? | Precision ↑ | ✅ Можно применять |
| Где v2.0 проигрывает? | Recall ↓ на основном классе | 🔴 Это неприемлемо, откатываемся |

**Метрика для GO:**
```
Совпадение v1.0 и v2.0 ≥ 95%
└─ Если < 95% → разбираемся в причине
```

**Шаг 4: Откат за < 5 минут (если нужно)**

```sql
-- Версия модели хранится как конфиг (в БД), не в коде
UPDATE ml_service_config 
SET model_version = 'v1.0' 
WHERE service_name = 'fraud_detection';

-- Сбросить cache
FLUSH redis;

-- Новые операции идут на v1.0 автоматически
-- Время отката: < 5 минут (включая перезагрузку)
```

### Примеры что может пойти не так в Shadow Mode

**🔴 Кейс 1: LightGBM version upgrade (4.2.0 → 4.3.0)**
```
Диагностический набор: PASS (precision одинаковая)
Shadow mode день 3: FAIL
└─ Обнаружено: новая версия 4.3.0 by-default использует другой алгоритм
   на маленьких деревьях (regression в их реализации)
└─ Решение: откатиться на 4.2.0, дождаться fix в 4.3.1
└─ Время отката: 4 минуты
```

**🔴 Кейс 2: Training data расширена (960 fraud → 2400 fraud)**
```
Диагностический набор: PASS (даже лучше)
Shadow mode день 5: FAIL
└─ Обнаружено: recall прошел хорошо на основных операциях
   Но на редких операциях (дробление) recall упал с 85% → 70%
└─ Причина: новые примеры дробления уменьшили precision на других классах
└─ Решение: переобучить с балансировкой классов (не просто scale_pos_weight)
└─ Время: 1 день переобучения
```

**✅ Кейс 3: Только config изменился (max_depth 7 → 8)**
```
Диагностический набор: PASS (даже улучшилось)
Shadow mode день 3: PASS
└─ Обнаружено: совпадение в 98% операций
├─ Precision: 71% → 72% ✅
└─ Recall: 84% → 85% ✅
└─ Решение: скидываем v2.0 в production (Blue-Green deployment)
└─ Время: < 1 минута переключения
```

---

## Feedback Loop: Использование исправлений эксперта

**Проблема:** Если просто добавлять исправления эксперта в training set:
- Закрепляем индивидуальные предпочтения
- Добавляем noise если эксперт ошибся
- Меняем distribution если новые примеры не репрезентативны

### Процесс (Zero-bias feedback loop)

```
1. Эксперт рассматривает операцию (Y_expert)
2. Система предложила (Y_ml)
3. Если Y_expert ≠ Y_ml:
   └─ Сохраняется в audit_log (но НЕ в training set!)

4. ЕЖЕНЕДЕЛЬНЫЙ разбор аналитиком:
   ├─ Сегментировать исправления по типам
   │  ├─ "Это одна и та же ошибка" (pattern)
   │  └─ "Это субъективное предпочтение эксперта X" (bias)
   ├─ Если 10+ однотипных ошибок:
   │  └─ Добавить в "переразметка очередь"
   └─ Если это субъективное:
       └─ Обсудить с AML Lead (может быть, гайдлайны неясны?)

5. ПЕРЕРАЗМЕТКА (if needed):
   ├─ Double-blind validation (два разных эксперта)
   ├─ Если согласились: добавить в training set
   └─ Версионировать: "training_set_v2.0" (был "training_set_v1.0")

6. ПЕРЕОБУЧЕНИЕ модели:
   ├─ На training_set_v2.0
   ├─ Запустить регрессионный прогон (Shadow mode)
   └─ Если OK → deployment
```

### Пример (закрытие feedback loop)

```
Week 1:
├─ Эксперт 5 раз заблокировал операции, которые ML одобрил
├─ Все 5 были "платежи в новую страну с новым контрагентом"
└─ Сохранено в audit_log

Week 2 анализ:
├─ Аналитик видит паттерн: "новая страна + новый контрагент"
├─ Это не субъективно, это РИСК
└─ Добавить правило в Rules engine: "новый контрагент + new country → YELLOW"

Result:
├─ Переобучить модель
├─ Добавить примеры в training set
└─ Следующая неделя: модель уже будет flagging "новая страна + новый контрагент" как risky
```

---

## Мониторинг (Production SLA)

| Метрика | Red Line | Check |
|---------|----------|-------|
| Inference latency p95 | > 150ms | Real-time |
| Service availability | < 99.5% | Every 5min |
| Precision (moving 1000) | < 60% | Every 1hour |
| Recall (moving 1000) | < 70% | Every 1hour |

**Алертинг:**
- 🔴 Red line → Page oncall
- 🟡 Yellow line → Slack

---

## Ожидаемый результат

| Метрика | Manual | ML | Выигрыш |
|---------|--------|----|---------| 
| AML hours/день | 200h | 40h | 80% automation |
| Latency | 1-5 мин | <100ms | 60-300x |
| Cost | 2M ₽/мес | 100K ₽/мес | 95% снижение |

---

**Next:** Component Design, Monitoring Plan, ADR Architecture.
