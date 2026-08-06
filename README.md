# SQL Commercial Growth Analytics

> **MoM commercial growth analysis** evaluating GTV, Net Revenue, Active Users, and AOV using SQL window functions & Python visualization.

---

##  Business Context

This project analyzes 19 months of fintech transaction data (Feb 2025 – Jul 2026) to understand **what's driving growth** is it user acquisition or monetization per user?

**Core finding:**
- Growth is **volume-driven** — user acquisition is the primary lever
- AOV is **flat** — users are not spending more per transaction
- Growth rate has **slow down** — from +188% (Feb 2025) to -13% (Jul 2026)
- **Jul 2026 contraction** warrants immediate investigation (seasonal vs structural)

---

## Metrics Defined

| Metric | Definition | Formula |
|--------|------------|---------|
| **GTV** | Gross Transaction Value | `sum (amount)` for successful transactions |
| **Net Revenue** | Estimated platform revenue | `GTV × 2.5%` (flat take rate assumed) |
| **Active Paying Users** | Unique users per month | `count (distinct user_id)` with ≥1 successful transaction |
| **AOV** | Average Order Value | `GTV / Total Orders` |
| **MoM Growth %** | Month-over-month change | `(Current - Previous) / Previous × 100%` |
| **ARPU** | Avg Revenue Per User | `GTV / Active Users` |

---

## Methodology

**3-CTE SQL Pipeline:**

```
raw_monthly → lag_monthly → final SELECT
```

1. **raw_monthly** — base aggregation (sum, count by month)
2. **lag_monthly** — achieve ratios + LAG for month
3. **final SELECT** — round for display, calculate MoM growth %

**Key decisions:**
- Rounding only at final output layer (preserve raw precision in calculations)
- `NULLIF` to prevent division-by-zero on first month
- Filter: `status = 'success'` only

---

## SQL Query

```sql
with raw_monthly as (select strftime('%Y-%m', created_at) AS month, count (trx_id) as total_orders, count (distinct (user_id)) as active_paying_user, sum (amount) as total_gtv_raw, sum (amount * 0.025) as total_net_revenue_raw
from transactions
where status ='success'
group by month
order by month asc
),

lag_monthly as (
select month, total_orders, active_paying_user, total_net_revenue_raw, total_gtv_raw,
(total_gtv_raw / total_orders) as aov_raw,
lag (total_gtv_raw) over (order by month) as prev_gtv_raw,
lag (total_net_revenue_raw) over (order by month) as prev_net_revenue,
lag (active_paying_user) over (order by month) as prev_active_pay_user,
lag (total_gtv_raw / total_orders) over (order by month) as prev_aov
from raw_monthly
)

select month,
  round(total_gtv_raw) as total_gtv,round(total_net_revenue_raw) as total_net_revenue, round(aov_raw, 2) as aov, active_paying_user, total_orders, round(((total_gtv_raw - prev_gtv_raw) * 100.0) / nullif(prev_gtv_raw, 0),2) as mom_gtv_growth_pct, round(((total_net_revenue_raw - prev_net_revenue) * 100.0) / nullif(prev_net_revenue, 0), 2) as mom_net_revenue_growth_pct, round(((active_paying_user - prev_active_pay_user) * 100.0) / nullif(prev_active_pay_user, 0), 2) as mom_user_growth_pct, round(((aov_raw - prev_aov) * 100.0) / nullif(prev_aov, 0), 2) as mom_aov_growth_pct
from lag_monthly
order by month asc;
```

---

## Python Visualization

```python
fig, ax = plt.subplots(figsize=(10, 5), facecolor='#FAFAFA')

ax.fill_between(df['month'], df['total_gtv'], alpha=0.15, color=BLUE)
ax.plot(df['month'], df['total_gtv'], marker='o', color=BLUE, linewidth=2.5, markersize=5, label='GTV')
ax.fill_between(df['month'], df['total_net_revenue'], alpha=0.2, color=GREEN)
ax.plot(df['month'], df['total_net_revenue'], marker='s', color=GREEN, linewidth=2, markersize=4, label='Net Revenue')

ax.set_title('GTV vs Net Revenue ', fontweight='bold', color='#1A3A5C', fontsize=12, pad=12)
ax.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x/1e6:.1f}M'))
ax.legend(fontsize=9)
ax.tick_params(axis='x', rotation=45)

plt.tight_layout()
plt.savefig('01_gtv_vs_net_revenue.png', dpi=200, bbox_inches='tight', facecolor='#FAFAFA')
plt.show()
```
<img width="989" height="490" alt="image" src="https://github.com/user-attachments/assets/ffd3f26c-5838-41b3-a1c0-156fc190a9d5" />


---

## Next Steps (Prioritized by Impact × Effort)

| Priority | Initiative | Effort | Impact | Status |
|----------|------------|--------|--------|--------|
| 🔴 **1** | **Cohort Retention Analysis** | 2–3h | HIGH | Planned |
| 🟡 **2** | **Jul Contraction Root Cause** | 1–2h | HIGH | Planned |
| 🟡 **3** | **Segment Breakdown by Product** | 1h | MEDIUM | Planned |
| 🟢 **4** | **ARPU Calculation** | 30m | MEDIUM | ✅ Added to viz |
| 🟢 **5** | **3-Month Forecast** | 30m–2h | MEDIUM | Planned |

### Priority 1 — Cohort Retention

Real question: Are users *staying*, or is acquisition hiding churn?

```sql
WITH first_transaction AS (
  SELECT user_id,
         strftime('%Y-%m', MIN(created_at)) AS cohort_month
  FROM transactions
  WHERE status = 'success'
  GROUP BY user_id
),
monthly_activity AS (
  SELECT user_id,
         strftime('%Y-%m', created_at) AS activity_month
  FROM transactions
  WHERE status = 'success'
  GROUP BY user_id, activity_month
)
SELECT
  ft.cohort_month,
  ma.activity_month,
  COUNT(DISTINCT ma.user_id) AS active_users
FROM first_transaction ft
LEFT JOIN monthly_activity ma ON ft.user_id = ma.user_id
  AND ma.activity_month >= ft.cohort_month
GROUP BY ft.cohort_month, ma.activity_month
ORDER BY ft.cohort_month, ma.activity_month
```

### Priority 2 — Jul 2026 Root Cause

Is the Jul contraction seasonal or structural?

```sql
-- Step 1: YoY comparison (Jul only)
SELECT strftime('%Y-%m', created_at) AS month,
       COUNT(DISTINCT user_id) AS users,
       SUM(amount) AS gtv
FROM transactions
WHERE strftime('%m', created_at) = '07'
GROUP BY month ORDER BY month;

-- Step 2: Daily breakdown in Jul 2026
SELECT DATE(created_at) AS day,
       COUNT(*) AS transactions,
       SUM(amount) AS daily_gtv
FROM transactions
WHERE strftime('%Y-%m', created_at) = '2026-07'
GROUP BY day ORDER BY day;
```

### Priority 3 — Segment by Product Category

Is "AOV flat" true across all products, or masked by mix shift?

```sql
SELECT strftime('%Y-%m', created_at) AS month,
       product_category,
       COUNT(DISTINCT user_id) AS users,
       COUNT(*) AS orders,
       ROUND(SUM(amount) / COUNT(*), 2) AS aov,
       ROUND(SUM(amount)) AS gtv
FROM transactions
WHERE status = 'success'
GROUP BY month, product_category
ORDER BY month, product_category;
```

---

## Key Insights & Interpretation

### What the data says

- **Volume-driven growth** — User acquisition is the primary growth lever; AOV is flat
- **Decelerating trend** — Growth cooling from +188% (Feb 2025) to -13% (Jul 2026)
- **Jul 2026 contraction** — All metrics negative; warrants investigation

### What it might mean

- Flat AOV + declining ARPU → Users transacting **less frequently** (retention problem)
- Flat AOV + flat ARPU → User mix shifting (new users ≈ old users in spend)
- Decelerating user growth → Either CAC rising or market maturing


---

## Tech Stack

- **SQL**: SQLite — 3-CTE pipeline with LAG window functions
- **Python**: Pandas, Matplotlib (visualization)
- **Environment**: Google Colab
- **Data**: Synthetic fintech transaction dataset (19 months)

---

## Repository Structure

```
sql-commercial-growth-analytics/
├── README.md              ← This file (analysis + viz + next steps)
├── analysis.ipynb         ← Full notebook (SQL + Python)
├── commercial_growth_dashboard.png  ← Output visualization
```
---

*Analysis by Mahesworo Langgeng Wicaksono · [LinkedIn](https://linkedin.com/in/mahesworo-langgeng-wicaksono-897483148)*
