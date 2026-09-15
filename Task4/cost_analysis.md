# Анализ затрат: Full TCO

**Инициатива:** #1 Выявление подозрительных операций  
**Baseline (Manual):** 2M ₽/месяц (25 AML экспертов)  
**Target (ML):** ?  

---

## Инфраструктура (Compute & Storage)

| Компонент | Кол-во | Стоимость | Всего |
|-----------|--------|----------|-------|
| **ML Service (FastAPI pods)** | 3 replicas | 50K/месяц | 150K |
| **Load Balancer (Nginx)** | 1 | 10K/месяц | 10K |
| **Redis Cache** | 1 | 5K/месяц | 5K |
| **Postgres DB** | 1 master + 1 standby | 20K/месяц | 40K |
| **API Gateway (Kong)** | 1 | 10K/месяц | 10K |
| **Monitoring (Prometheus+Grafana)** | 1 | 5K/месяц | 5K |
| **Storage (S3 backups)** | 100GB/month | 1K | 1K |
| **Network & Egress** | ~1TB/month | 5K/месяц | 5K |

**Итого Compute:** ~226K ₽/месяц (при deployment на AWS/GCP)  
**При on-prem:** ~150K ₽/месяц (own infra)

---

## Разработка и поддержка (People)

### Поддержка системы (ML-focused)

| Роль | Кол-во | Зарплата | Всего |
|------|--------|----------|-------|
| **ML Engineer** | 0.5 FTE | 150K ₽/месяц | 75K |
| **DevOps/Platform** | 0.25 FTE | 120K ₽/месяц | 30K |
| **Compliance Officer** (DPO monitoring) | 0.25 FTE | 120K ₽/месяц | 30K |

**Итого ML Support:** 135K ₽/месяц

### Эксперты AML (Expert Review Team)

| Роль | Кол-во | Зарплата | Всего |
|------|--------|----------|-------|
| **Senior AML Expert** | 2 | 150K ₽/месяц | 300K |
| **Junior AML Expert** | 8 | 100K ₽/месяц | 800K |

**Итого Expert Team:** 1.1M ₽/месяц

⚠️ **КЛЮЧЕВОЕ РАЗЛИЧИЕ:** 
- Baseline (manual): 25 экспертов = 2M ₽
- С ML: 10 экспертов = 1.1M (эксперты review ML scores)
- Но они ОСТАЮТСЯ в команде: система не удаляет их, переводит на качественный контроль

**Итого People:** ~1.235M ₽/месяц

---

## Лицензии и third-party

| Сервис | Стоимость | Комментарий |
|--------|----------|-----------|
| **LightGBM** | Free | Open source |
| **Python + Libraries** | Free | Open source |
| **Prometheus + Grafana** | Free | Open source |
| **PagerDuty (on-call alerts)** | 10K ₽/месяц | For redundancy |
| **Regulatory/DPO Tools** | 10K ₽/месяц | Audit logs, compliance reporting |

**Итого Licenses:** ~20K ₽/месяц

---

## Первоначальные инвестиции (One-time)

| Item | Стоимость | Комментарий |
|------|----------|-----------|
| **Infrastructure setup** | 200K | Dev/Staging/Production env |
| **CI/CD pipeline** | 100K | Jenkins/GitLab/GitHub Actions |
| **Model training infra** | 150K | GPU for retraining |
| **Documentation & training** | 50K | Oncall playbooks, runbooks |
| **Security audit** | 100K | Compliance review |

**Итого One-time:** ~600K ₽

---

## Сравнение: Manual vs ML

### Baseline (текущее: ручная обработка)

| Компонент | Стоимость | Комментарий |
|-----------|-----------|-----------|
| **25 AML Экспертов** | 2M ₽/месяц | Полный manual review всех операций |
| **Обучение** | 100K/месяц | Continuous training программа |
| **Tools** | 50K/месяц | Legacy система управления (Salesforce/1C) |
| **Audit/Compliance** | 50K/месяц | Контроль качества работы экспертов |
| **Overhead** | 50K/месяц | Управление, HR, office space |

**ИТОГО BASELINE:** 2.25M ₽/месяц

### ML Solution (новое)

| Компонент | Стоимость | Комментарий |
|-----------|-----------|-----------|
| **Compute & Infrastructure** | 226K ₽/месяц | ML service + DB + monitoring |
| **ML Support Team** | 135K ₽/месяц | 0.5 ML Engineer + 0.25 DevOps + 0.25 DPO |
| **Expert Team (10 AML experts)** | 1.1M ₽/месяц | Новая роль: review ML scores + escalations |
| **Licenses & Tools** | 20K ₽/месяц | DPO tools + alerts + regulatory logs |
| **Retroactive Разметка** | 180K ₽/месяц | Амортизированная (960→2400 fraud примеров) |
| **Overhead** | 50K/месяц | Management, training, compliance |

**ИТОГО ML SOLUTION:** 1.711M ₽/месяц

### Чистая экономия

| Метрика | Значение |
|---------|----------|
| **Месячная экономия** | 2.25M - 1.711M = **539K ₽/месяц** |
| **Годовая экономия** | 539K × 12 = **6.468M ₽/год** |
| **Payback period** | 600K (setup) / 539K = **1.1 месяца** (~5 недель) |
| **Year 1 ROI** | (6.468M - 0.6M) / 0.6M = **977%** |

⚠️ **Важно:** Это консервативный расчет. Реальная экономия может быть выше:
- Если мы ловим БОЛЬШЕ fraud → меньше потерь для банка
- Если эксперты могут обрабатывать более сложные case → качество выше
- Если мы автоматизируем рутину → эксперты думают о стратегии

---

## Сценарии стоимости

### Сценарий 1: Base Case (Cloud + 10 Experts)

```
Compute/Infra:  226K ₽/месяц
ML Support:     135K
Expert Team:   1.1M
Licenses:        20K
Retroactive:    180K
Overhead:        50K
────────────────────────
TOTAL OPEX:    1.711M ₽/месяц

Baseline:      2.25M ₽/месяц
Savings:       539K ₽/месяц
Year 1 ROI:    977% ✅
```

### Сценарий 2: On-Prem (own infra, lower compute cost)

```
Compute/Infra:  150K ₽/месяц
ML Support:     135K
Expert Team:   1.1M
Licenses:        20K
Retroactive:    180K
Overhead:        50K
────────────────────────
TOTAL OPEX:    1.635M ₽/месяц

Savings:       615K ₽/месяц (+14% vs Cloud)
Year 1 ROI:    1025% ✅
```

### Сценарий 3: Worst-case (need more experts or compute scaling)

```
Compute/Infra:  500K ₽/месяц (10x pods)
ML Support:     200K (2 FTE)
Expert Team:   1.3M (12 experts, peak load)
Licenses:        50K
Retroactive:    200K
Overhead:       100K
────────────────────────
TOTAL OPEX:    2.35M ₽/месяц

Savings:      -100K (LOSS if baseline is fixed)
BUT: Baseline also scales (more experts needed)
Actual benefit: Better quality with same cost
```

### Сценарий 4: Best-case (higher automation rate, fewer experts)

```
If ML catches 90% (vs 70% baseline):
├─ Can reduce expert team to 6 people
├─ Expert Team: 700K ₽/месяц (instead of 1.1M)
└─ TOTAL OPEX: 1.4M ₽/месяц

Savings:       850K ₽/месяц (37% reduction!)
Year 1 ROI:    1317% 🎉
```

---

## Риски к стоимости

| Риск | Вероятность | Влияние | Смягчение |
|------|------------|---------|-----------|
| **Infra underestimated** | Medium | +50K/месяц | 10% buffer built in |
| **Model retraining frequent** | Low | +100K/месяц (GPU) | Only if drift detected |
| **Compliance costs spike** | Low | +200K/месяц | Proactive audits |
| **Scaling needed (10x)** | Low | +300K/месяц | Auto-scale available |

---

## Финальная стоимость

### Год 1

| Item | Cost | Notes |
|------|------|-------|
| **Setup (one-time)** | 600K ₽ | Infra + CI/CD + security audit |
| **Operations (monthly)** | 1.711M ₽/месяц | Base case: Cloud + 10 experts |
| **Operations (12 months)** | 20.532M ₽ | 1.711M × 12 |
| **TOTAL Year 1** | **21.132M ₽** | |

### Год 2 и далее

| Item | Cost | Notes |
|------|------|-------|
| **Operations (monthly)** | 1.711M ₽/месяц | Same as Year 1 (no setup cost) |
| **Operations (12 months)** | 20.532M ₽ | Annual operating cost |

### Анализ экономии

**Baseline (все ручное):** 2.25M ₽/месяц = 27M ₽/год

| Period | Total Cost (ML) | Baseline Cost | Net Benefit |
|--------|---|---|---|
| **Year 1** | 21.132M | 27M | **+5.868M ₽** (22% сокращение) |
| **Year 2** | 20.532M | 27M | **+6.468M ₽** (24% сокращение) |
| **3-year Total** | 62.196M | 81M | **+18.804M ₽** (23% сокращение) |

**Payback period:** 600K (setup) / 539K (monthly savings) = **1.1 месяца**

После первого месяца система окупается и начинает приносить чистую прибыль ✅

---

**Next:** Pilot Planning + Go/No-Go criteria (как проверим на реальных данных).
