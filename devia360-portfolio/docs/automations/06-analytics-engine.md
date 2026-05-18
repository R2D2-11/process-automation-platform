# 06 — Analytics & Historical Intelligence

> How I gave a manufacturing plant its first real ability to analyze performance over time — replacing "react after failure" with "detect drift before it becomes a problem."

## The problem

Before DEVIA, the plant had data — it just couldn't use it effectively.

**RFT (Right First Time) tracking:** This metric measures what percentage of production orders pass quality control on the first attempt, without needing adjustments. It was tracked using a helper column in the Excel-Kanban file with a simple Yes/No value. The metric itself worked, but the surrounding analysis was broken:
- Time-based breakdowns (daily, weekly, monthly trends) required manual filtering
- Cross-referencing RFT with product type, operator, or shift was impractical
- Any analysis beyond the basic number required exporting, cleaning, and re-processing data

**Quality parameter trends:** Test results (viscosity, pH, density, etc.) were stored as values in Excel columns. To spot a trend — like a parameter slowly drifting toward its limit over several batches — someone would have to:
1. Open the right Excel file
2. Find the right column
3. Visually scan rows of numbers
4. Hope they noticed the pattern

**Nobody did this proactively.** The team only looked at parameters after a failure — by which time the damage was done.

**Cycle time analysis:** How long does a production order take? Which stages are bottlenecks? Before DEVIA, these questions were unanswerable. Time wasn't tracked per stage.

## The solution

### SINTEGRA: the historical intelligence module

SINTEGRA (Analítica e Historial) provides a consolidated view of all completed production orders with three analytical capabilities:

### 1. Historial Maestro — Production history

Every closed OP is stored with:
- Complete metadata (product, lot, operator, dates, order type)
- **Time spent in each of the 7 Kanban stages** — enabling cycle time analysis
- Associated weighing sheets and lab results
- RFT status (pass on first attempt: yes/no)
- Tonnage processed

Users can filter by product, lot, operator, date range, and more. This is the first time anyone at the plant could answer questions like:

- "What's our average cycle time for Product X?"
- "Which stage has the most delays?"
- "How does Operator A's throughput compare to Operator B's?"
- "What percentage of orders for Customer Y required adjustments?"

### 2. RFT calculation with granularity

RFT is now calculated automatically with breakdowns by:

```
RFT by day:    ████████░░  80%  (Mon)
               ██████████  100% (Tue)
               ██████░░░░  60%  (Wed)

RFT by week:   Week 1: 85%  │  Week 2: 78%  │  Week 3: 92%

RFT by month:  Jan: 82%  │  Feb: 87%  │  Mar: 91%  (improving trend ✓)

RFT by product: Product A: 94%  │  Product B: 71%  (investigate ⚠️)
```

This transforms RFT from a single number ("our RFT this quarter was 83%") into a diagnostic tool that reveals when, where, and why quality issues happen.

### 3. Parameter trend charts

For each product and each test parameter, DEVIA generates historical trend charts showing how values have evolved across batches:

```
Viscosity — Product X (last 20 batches)
                                              ┌─ Upper Limit ─ ─ ─ ─
    ●                                         │
        ●   ●                          ●      │
            ●  ●   ●   ●        ●   ●        │
                       ●   ●  ●              │
                                   ●          │
                                              │
                                              └─ Lower Limit ─ ─ ─ ─
    Batch: 1  2  3  4  5  6  7  8  9 10 11 12
    
    → Values trending upward. Not failing yet, but approaching upper limit.
    → Investigate raw material source or process parameters.
```

This covers all three analysis stages:
- **Raw material trends** — Are incoming materials degrading in quality over time?
- **Fabrication trends** — Is the process drifting?
- **Finished product trends** — Are we getting closer to spec limits?

**Before:** "We failed viscosity on Batch 47." (Reactive)
**After:** "Viscosity has been trending up for the last 8 batches. Let's investigate before it fails." (Proactive)

## Impact

| Metric | Before | After |
|--------|--------|-------|
| RFT analysis | Basic Yes/No column, manual quarterly review | Automated with day/week/month/product granularity |
| Cycle time per stage | Not tracked | Measured for every OP, filterable |
| Quality trend analysis | None — react after failure only | Historical trend charts, early drift detection |
| Time to generate a performance report | Hours of manual Excel work | Seconds (filter + view) |
| Data-driven decision making | Anecdotal ("it feels like we're slower") | Evidence-based ("Stage 3 averages 2.3 days for Product X") |

## Why this matters beyond manufacturing

Any platform processing structured submissions at scale needs analytics to improve:

- **Triage performance metrics:** What's our average response time? Which report categories take longest? Where do reports get stuck? These are the same questions as "which Kanban stage is the bottleneck?"
- **Quality trend monitoring:** Are we getting more spam this month? Is a particular program generating more disputes? Are false-duplicate rates increasing? Same pattern as parameter drift detection.
- **Contributor analytics:** Which analysts have the highest resolution rate? Where are the training gaps? Same as RFT by operator.

The difference between a reactive operation and a proactive one is the same regardless of industry: **can you see the trend before it becomes a crisis?**

---

← [Previous: Data Validation Engine](05-data-validation.md) · ← [Back to README](../../README.md)
