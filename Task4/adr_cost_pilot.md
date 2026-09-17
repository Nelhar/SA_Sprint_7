# ADR: Стоимость и стратегия пилота

**Статус:** ✅ ПРИНЯТО
**Дата:** 2024-10-16
**Источник цифр:** cost_analysis.md + roi_calculation.md (единая база — TCO **1.711M ₽/месяц**)

---

## Проблема

Как обосновать инвестицию в ML систему, если:

1. **Заметная стоимость** (1.711M ₽/месяц операционных затрат, из них 1.1M — эксперты)
2. **Неопределённость** на производстве (может ли модель работать на реальных данных?)
3. **Риск отката** (что если pilot не сработает?)
4. **Бизнес-согласование** (как убедить CFO потратить деньги, если ROI умеренный — 28%, а не сотни процентов?)

---

## Решение: Progressive Rollout с честно пересчитанным ROI

### 1. Стоимость (Cost Analysis)

**Baseline (Manual):** 2.25M ₽/месяц (25 AML экспертов × 80K = 2M + overhead 250K)

**New (ML):** 1.711M ₽/месяц (10 экспертов, из них 2 senior + 8 junior = 1.1M, плюс инфраструктура, ML-support, лицензии, retroactive разметка и overhead)

**Net savings:** 539K ₽/месяц (~24% снижение затрат)

**One-time investment:** 600K ₽ (setup + инфраструктура + обучение)

**Payback period:** 1.1 месяца (~33 дня) ✅

---

### 2. ROI Analysis

**Методология:** ROI = Net Benefit / Total Cost за период, где Total Cost = one-time setup + 12 месяцев эксплуатации. Расчёт консервативный — включает ТОЛЬКО подтверждённую экономию (замещение ручного труда), без спекулятивной денежной оценки "дополнительно пойманного fraud" (это качественный upside, см. roi_calculation.md).

**Year 1:**
- Total cost: 600K (setup) + 1.711M × 12 = 21.132M ₽
- Benefit (baseline больше не оплачивается): 27M ₽
- **Net profit:** 5.868M ₽
- **ROI:** 28%

**Year 2+:**
- Total cost: 20.532M ₽ (без setup)
- Benefit: 27M ₽
- **Net profit:** 6.468M ₽/год
- **ROI:** 32%

**Дополнительный (не включённый в ROI) эффект:** Recall вырастает с 67% до 84% — банк ловит на ~408 больше подозрительных операций/месяц. Денежная оценка будет уточнена по итогам пилота.

**Критичное допущение:** весь бизнес-кейс держится на том, что 10 экспертов справляются с новым потоком задач (20% операций, Yellow-класс). Если пилот покажет, что нужно 15 экспертов, ROI Года 1 падает почти до нуля (см. Sensitivity Analysis в roi_calculation.md).

---

### 3. Pilot Strategy (Phased Approach)

**Phase 1:** Synthetic data (Week 1-2) → Validation of system
**Phase 2:** Real data 10% traffic (Week 3-6) → Parallel testing vs rules
**Phase 3:** Gradual ramp (Week 7-8) → 20% → 50% → 80% → 100%

**Rollback:** If any red metrics → automatic fallback to rules

**Decision gate:** Post-pilot review, go/no-go decision (подробные количественные критерии — в pilot_planning.md)

---

## Архитектурные решения

### 1. Cost Optimization

**Выбор:** ✅ **Cloud + On-Prem Hybrid** (optimize both)

| Компонент | Where | Cost | Rationale |
|-----------|-------|------|-----------|
| ML Service | Cloud (AWS) | 150K ₽/месяц | Scales easily |
| Database | On-Prem | 40K ₽/месяц | Data sovereignty |
| Cache | Cloud | 5K ₽/месяц | High availability |
| Backups | S3 | 5K ₽/месяц | Low cost storage |

**Выигрыш:** Комбинация даёт компонент "Compute/Infra" на уровне 226K ₽/месяц (Cloud) или 150K ₽/месяц (On-Prem), compliance со storage.

---

### 2. Expert Reallocation (NOT Layoff)

**Выбор:** ✅ **Upskill + Redeploy 15 из 25 экспертов**, 10 остаются на AML-review

| Role | Count | Future |
|------|-------|--------|
| AML Manual Review (hard cases + Yellow-класс) | 25 | **10** (2 senior + 8 junior) |
| AML Model Trainer | — | 5 (new role: improve model) |
| Compliance Officer | — | 3 (enhanced role: QA) |
| Risk Analyst | — | 5 (new role: thresholds, rules) |
| Onboarding/Training | — | 2 (customer education) |
| **Total headcount** | 25 | 25 (repurposed) |

**Benefit:** Zero layoffs, upskilled team, better career growth. ФОТ AML-review функции сокращается с 2M до 1.1M ₽/месяц (экономия 900K ₽/месяц), остальные 15 экспертов переходят на новые роли с отдельным бюджетом (не входит в TCO этой инициативы).

---

### 3. Fallback Strategy

**Выбор:** ✅ **Rules Engine as Fallback** (not just shutdown)

If ML service fails:
```
ML down → Fallback to RULES ENGINE (already designed in Task 3)
         → Process continues at 90% quality
         → Experts notified for manual review if needed
         → No operations lost
```

**Implication:** Zero business downtime, even if ML fails.

---

## Допущения (Assumptions)

| # | Допущение | Статус | Risk | Mitigation |
|---|-----------|--------|------|-----------|
| **A1** | Model performs on real data (71% precision, 84% recall) | ✅ Verified in Task 2 | Low | Pilot validation (week 3-6) |
| **A2** | 10 экспертов достаточно для Yellow-класса (20% потока) | 📋 TBD | **High** (ROI около нуля если нужно 15) | Gradual ramp (week 7-8), явный Go/No-Go критерий |
| **A3** | Infra costs estimated correctly (226K ₽/месяц) | 📋 TBD | Medium | +20% buffer, auto-scale |
| **A4** | Payback в 1.1 месяца достижим | 📋 TBD | Low | Consistent с cost_analysis.md |
| **A5** | Fallback rules work if ML fails | ✅ Designed | Low | Tested in phase 1 |
| **A6** | Compliance approves model (DPO sign-off) | ⏳ Pending | High | Pilot validation helps |

---

## Риски и смягчение

| Риск | Prob | Impact | Mitigation |
|------|------|--------|-----------|
| **Model underperforms in prod** | Medium | High | Progressive rollout + fallback |
| **Нужно 15 экспертов вместо 10** | Medium | **High** (ROI ≈ 0) | Пилот Phase 2-3 должен подтвердить нагрузку до полного запуска |
| **Infra costs overrun 2x** | Low | Medium | ROI падает до 13%, но остаётся положительным |
| **Experts can't handle new workflow** | Low | Medium | Training + gradual ramp |
| **Compliance blocks due to regulation** | Low | High | Fallback to rules (compliant) |
| **Data quality worse than expected** | Low | Low | Model retraining after pilot |

---

## Decision Criteria (Go/No-Go)

**After pilot, check:**

| Criterion | Target | Go condition |
|-----------|--------|--------------|
| Latency p95 | <150ms | Achieved consistently |
| Error rate | <1% | No critical errors |
| Precision | >60% | Better than rules (71% target) |
| Recall | >70% | Better than manual (84% target) |
| Cost | ≤1.711M ₽/месяц (±20% buffer до ~2.05M) | Within budget |
| Экспертная нагрузка | 10 экспертов справляются с Yellow-потоком | Не требуется расширение штата |
| Fallback works | Fast recovery | <1 min rollback |
| Experts satisfied | >80% approval | Team consensus |
| Compliance OK | Written sign-off | DPO approval |

**If all ✅ → PRODUCTION DEPLOYMENT**
**If any ❌ → WAIT, DEBUG, RETRY (or abort)**

Полные количественные GO / NO-GO / PIVOT критерии — в pilot_planning.md.

---

## Business Case Summary

| Metric | Value | Timeframe |
|--------|-------|-----------|
| **Payback period** | 1.1 месяца (~33 дня) | ~ 5 недель |
| **Year 1 ROI** | 28% | 12 months |
| **Year 2+ ROI** | 32% | 12 months |
| **Cost avoidance** | 539K ₽/месяц | Ongoing |
| **Process improvement** | 60-300x faster | Immediate |
| **Fraud detection** | Recall +17 п.п. (67%→84%) | Ongoing, денежный эффект не оценён |
| **Risk reduction** | Objective decisions | Immediate |

**Verdict:** ✅ **APPROVAL RECOMMENDED** (умеренный, но защитимый бизнес-кейс; критично зависит от подтверждения A2 в пилоте)

---

**Next:** Final README (everything in one place).
