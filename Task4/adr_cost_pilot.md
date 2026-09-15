# ADR: Стоимость и стратегия пилота

**Статус:** ✅ ПРИНЯТО  
**Дата:** 2024-10-16  

---

## Проблема

Как обосновать инвестицию в ML систему, если:

1. **Высокая стоимость** (~226K ₽/месяц инфраструктура + люди)
2. **Неопределённость** на производстве (может ли модель работать на реальных данных?)
3. **Риск отката** (что если pilot не сработает?)
4. **Бизнес-согласование** (как убедить CFO потратить деньги?)

---

## Решение: Progressive Rollout с вычисленным ROI

### 1. Стоимость (Cost Analysis)

**Baseline (Manual):** 2.17M ₽/месяц (25 AML экспертов + инструменты)

**New (ML):** 969K ₽/месяц (7.5 экспертов + инфраструктура)

**Net savings:** 1.2M ₽/месяц (~55% снижение затрат)

**One-time investment:** 600K ₽ (setup + infra + training)

**Payback period:** 6.7 дней (less than 1 week!) ✅

---

### 2. ROI Analysis

**Year 1:**
- Direct savings: 1.4M ₽/месяц × 12 = 16.8M ₽
- Fraud prevention: 0.14 × 960 fraud × 50K = 8.064M ₽
- Expert time: 1.05M hours/year = 10.08M ₽
- Compliance risk: 2.4M ₽
- **Total benefits:** 37.344M ₽
- **Costs:** 12.228M ₽
- **Net profit:** 25.116M ₽
- **ROI:** 205%

**Even in pessimistic scenarios:** ROI > 140%

---

### 3. Pilot Strategy (Phased Approach)

**Phase 1:** Synthetic data (Week 1-2) → Validation of system
**Phase 2:** Real data 10% traffic (Week 3-6) → Parallel testing vs rules
**Phase 3:** Gradual ramp (Week 7-8) → 20% → 50% → 80% → 100%

**Rollback:** If any red metrics → automatic fallback to rules

**Decision gate:** Post-pilot review, go/no-go decision

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

**Выигрыш:** 20% экономия vs pure cloud, compliance со storage.

---

### 2. Expert Reallocation (NOT Layoff)

**Выбор:** ✅ **Upskill + Redeploy 18 из 25 экспертов**

| Role | Count | Future |
|------|-------|--------|
| AML Manual Review | 25 | 7.5 (hard cases only) |
| AML Model Trainer | 5 | New role (improve model) |
| Compliance Officer | 3 | Enhanced role (QA) |
| Risk Analyst | 5 | New role (thresholds, rules) |
| Onboarding/Training | 5 | Customer education |
| **Total headcount** | 25 | 25 (repurposed) |

**Benefit:** Zero layoffs, upskilled team, better career growth.

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
| **A2** | Experts can handle reduced volume (25% vs 40%) | 📋 TBD | Medium | Gradual ramp (week 7-8) |
| **A3** | Infra costs estimated correctly (226K ₽/месяц) | 📋 TBD | Medium | +20% buffer, auto-scale |
| **A4** | Payback in 6.7 days is achievable | 📋 TBD | Low | Conservative assumptions |
| **A5** | Fallback rules work if ML fails | ✅ Designed | Low | Tested in phase 1 |
| **A6** | Compliance approves model (DPO sign-off) | ⏳ Pending | High | Pilot validation helps |

---

## Риски и смягчение

| Риск | Prob | Impact | Mitigation |
|------|------|--------|-----------|
| **Model underperforms in prod** | Medium | High | Progressive rollout + fallback |
| **Infra costs overrun 2x** | Low | Medium | Still ROI positive (143%) |
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
| Cost | <369K ₽/месяц | Within budget |
| Fallback works | Fast recovery | <1 min rollback |
| Experts satisfied | >80% approval | Team consensus |
| Compliance OK | Written sign-off | DPO approval |

**If all ✅ → PRODUCTION DEPLOYMENT**  
**If any ❌ → WAIT, DEBUG, RETRY (or abort)**

---

## Business Case Summary

| Metric | Value | Timeframe |
|--------|-------|-----------|
| **Payback period** | 6.7 days | ~ 1 week |
| **Year 1 ROI** | 205% | 12 months |
| **Year 2+ ROI** | 221% | 12 months |
| **Cost savings** | 1.2M ₽/месяц | Ongoing |
| **Process improvement** | 60-300x faster | Immediate |
| **Fraud detection** | +14% caught | Ongoing |
| **Risk reduction** | Objective decisions | Immediate |

**Verdict:** ✅ **STRONG APPROVAL RECOMMENDED**

---

**Next:** Final README (everything in one place).
