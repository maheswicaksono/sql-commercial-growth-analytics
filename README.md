# sql-commercial-growth-analytics
MoM commercial growth analysis evaluating GTV, net revenue, active users, and AOV using SQL window functions &amp; Python visualization.

# Fintech Business Performance Analysis

Analysis of fintech transaction data (19 months) to understand growth drivers: user acquisition vs. monetization per user.

## Key Findings

- **Volume-driven growth**: User acquisition is the primary growth lever; AOV is flat
- **Decelerating trend**: Growth rate cooling from 188% (Feb 2025) to -13% (Jul 2026)
- **Jul 2026 contraction**: All metrics negative — warrants investigation (seasonal vs. structural)
- **Take rate assumption**: Net revenue modeled as flat 2.5% of GTV

## Data & Metrics

| Metric | Definition |
|--------|-----------|
| GTV | Sum of successful transaction amounts |
| Net Revenue | GTV × 2.5% (flat take rate) |
| Active Users | Distinct users per month |
| AOV | GTV ÷ transaction count |
| MoM Growth | (Current - Previous) / Previous × 100% |

## Technical

- **Query**: 3-CTE pipeline (raw aggregation → derivation with LAG → final select)
- **Language**: SQL (SQLite)
- **Visualization**: Python (Pandas, Matplotlib, Seaborn)

Key decision: Rounding only at final output, preserve raw precision in calculations.

## Visualizations

1. **macro_performance_trend.png** — GTV vs Net Revenue growth trajectory
   - Note: Perfectly correlated due to flat take rate assumption

2. **growth_driver_divergence.png** — User growth vs AOV growth
   - Blue line (user growth): Consistently positive but declining
   - Red line (AOV growth): Volatile, centered around 0%

## Assumptions

- Take rate: Flat 2.5% (real business varies by product/payment method)
- Filter: Only `status = 'success'` transactions
- User count: DISTINCT per month (repeat users counted once)
- No seasonality adjustment or cohort retention modeling

## What I'd Do Next

If extending this analysis:
1. Cohort retention table (are 2025 users stickier than 2026?)
2. Segment by product category (which products drive growth?)
3. Is Jul contraction seasonal? (need 2-3 years of data to confirm)

## Files

- `README.md` — This file
- `improved_query.sql` — SQL query with inline comments
- `macro_performance_trend.png` — Chart 1
- `growth_driver_divergence.png` — Chart 2

## How to Run

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect('your_database.db')
df = pd.read_sql_query(open('improved_query.sql').read(), conn)
print(df)
```

---

**Status**: ✅ Complete  
**Dataset**: 2025-01 to 2026-07 (synthetic)
