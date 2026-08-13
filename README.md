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
with raw_monthly as (select strftime('%Y-%m', created_at) AS month, count (trx_id) as total_orders,
count (distinct (user_id)) as active_paying_user, sum (amount) as total_gtv_raw,
sum (amount * 0.025) as total_net_revenue_raw
from transactions
where status ='success'
group by month
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
round(total_gtv_raw) as total_gtv,round(total_net_revenue_raw) as total_net_revenue, round(aov_raw, 2) as aov,
active_paying_user, total_orders, round(((total_gtv_raw - prev_gtv_raw) * 100.0) / nullif(prev_gtv_raw, 0),2) as mom_gtv_growth_pct, round(((total_net_revenue_raw - prev_net_revenue) * 100.0) / nullif(prev_net_revenue, 0), 2) as mom_net_revenue_growth_pct,
round(((active_paying_user - prev_active_pay_user) * 100.0) / nullif(prev_active_pay_user, 0), 2) as mom_user_growth_pct,
round(((aov_raw - prev_aov) * 100.0) / nullif(prev_aov, 0), 2) as mom_aov_growth_pct
from lag_monthly
order by month asc;
```

---

## Data Visualization

#### GTV vs. Net Revenue Trajectory
<img width="989" height="490" alt="image" src="https://github.com/user-attachments/assets/ffd3f26c-5838-41b3-a1c0-156fc190a9d5" />

* Both metrics track together (flat take rate assumption). GTV peaks around mid-2025 at 50M+, then stabilizes around 20-30M in 2026. Perfect correlation expected because net revenue = GTV × 2.5%.

#### MoM GTV Growth
<img width="990" height="490" alt="image" src="https://github.com/user-attachments/assets/7a7b13d1-32ca-4d76-879a-a50d364652a8" />

* MoM GTV shows clear phases: Revenue growth cooled down significantly from an early boom (+188% in Feb 2025) to a steady pace (+3–30%), eventually hitting its first drop in July 2026 (-13%)

#### Active Paying Users Trend
<img width="989" height="490" alt="image" src="https://github.com/user-attachments/assets/6863bffe-26cc-464f-9753-a0556715f0ec" />

* The active user base hit its peak in mid-2025 and has been slowly sliding, pointing to challenges in user acquisition and retention.

#### MoM User Growth Rate
<img width="990" height="490" alt="image" src="https://github.com/user-attachments/assets/bcc9b395-6f6a-4918-b558-3f2a89bf86ca" />

* User acquisition slowed sharply from +150% in early 2025 down to low digits (+5–15%) in 2026, eventually slipping into negative growth.

#### Average Order Value (AOV)
<img width="989" height="490" alt="image" src="https://github.com/user-attachments/assets/5e296e0a-828d-436c-873f-c828c5c333b4" />

* Average spending per order stayed completely flat over time, meaning users aren't buying higher-value items.

#### Growth Driver Gap (Users vs. AOV)
<img width="990" height="490" alt="image" src="https://github.com/user-attachments/assets/d072e595-6f13-448c-b56d-a230735f43ed" />

* The gap between fluctuating user growth and flat AOV proves that growth was driven purely by bringing in more users, not by getting people to spend more.

#### Total Monthly Orders
<img width="989" height="490" alt="image" src="https://github.com/user-attachments/assets/3e1137ff-c17e-4a15-b287-6f4d704dbc76" />

* Total monthly orders fell right alongside active users, putting double pressure on overall revenue since spending per order didn't grow either.

#### Average Revenue Per User (ARPU)
<img width="989" height="490" alt="image" src="https://github.com/user-attachments/assets/502e37ae-8fc0-4aa9-92a9-f13aa138c3a8" />

*  A declining ARPU while basket size remains flat is a clear sign that existing users are buying less frequently.



---

## Next Steps (Prioritized by Impact × Effort)

| Priority | Initiative | Impact | Status |
|----------|------------|--------|--------|
| 🔴 | **Cohort Retention Analysis** | High | Planned |
| 🟡 | **Jul Contraction Root Cause** | High | Planned |
| 🟡 | **Segment Breakdown by Product** | Medium | Planned |
| 🟢 | **3-Month Forecast** | Medium | Planned |

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
