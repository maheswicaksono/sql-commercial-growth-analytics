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

Run this in Google Colab after executing the SQL query above:

```python
import pandas as pd
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import warnings
warnings.filterwarnings('ignore')

# ── Style ──
plt.rcParams.update({
    'font.family': 'sans-serif',
    'axes.spines.top': False,
    'axes.spines.right': False,
    'axes.grid': True,
    'grid.alpha': 0.3,
    'grid.linestyle': '--'
})

BLUE   = '#2E5D8E'
GREEN  = '#1D9E75'
ORANGE = '#EF9F27'
RED    = '#E74C3C'
PURPLE = '#534AB7'
GRAY   = '#95A5A6'

df = df_result.copy()
df['month'] = pd.to_datetime(df['month'])
df_plot = df.dropna(subset=['mom_gtv_growth_pct'])

fig = plt.figure(figsize=(18, 22))
fig.patch.set_facecolor('#FAFAFA')
fig.suptitle('Fintech Commercial Growth Analysis\nFeb 2025 – Jul 2026',
             fontsize=16, fontweight='bold', color='#1A3A5C', y=0.98)

# ── Chart 1: GTV & Net Revenue (absolute) ──
ax1 = fig.add_subplot(4, 2, 1)
ax1.fill_between(df['month'], df['total_gtv'], alpha=0.15, color=BLUE)
ax1.plot(df['month'], df['total_gtv'], marker='o', color=BLUE, linewidth=2.5, markersize=5, label='GTV')
ax1.fill_between(df['month'], df['total_net_revenue'], alpha=0.2, color=GREEN)
ax1.plot(df['month'], df['total_net_revenue'], marker='s', color=GREEN, linewidth=2, markersize=4, label='Net Revenue')
ax1.set_title('GTV vs Net Revenue (Absolute)', fontweight='bold', color='#1A3A5C')
ax1.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x/1e6:.1f}M'))
ax1.legend(fontsize=9)
ax1.tick_params(axis='x', rotation=45)

# ── Chart 2: MoM GTV Growth % ──
ax2 = fig.add_subplot(4, 2, 2)
colors_gtv = [RED if x < 0 else BLUE for x in df_plot['mom_gtv_growth_pct']]
bars = ax2.bar(df_plot['month'], df_plot['mom_gtv_growth_pct'], color=colors_gtv, alpha=0.8, width=20)
ax2.axhline(0, color='black', linewidth=0.8, linestyle='-')
ax2.set_title('MoM GTV Growth %', fontweight='bold', color='#1A3A5C')
ax2.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:.0f}%'))
ax2.tick_params(axis='x', rotation=45)
for bar, val in zip(bars, df_plot['mom_gtv_growth_pct']):
    if abs(val) > 5:
        ax2.text(bar.get_x() + bar.get_width()/2, bar.get_height() + (1 if val >= 0 else -4),
                f'{val:.0f}%', ha='center', va='bottom', fontsize=7, color='#333333')

# ── Chart 3: Active Paying Users ──
ax3 = fig.add_subplot(4, 2, 3)
ax3.fill_between(df['month'], df['active_paying_user'], alpha=0.15, color=PURPLE)
ax3.plot(df['month'], df['active_paying_user'], marker='o', color=PURPLE, linewidth=2.5, markersize=5)
ax3.set_title('Active Paying Users per Month', fontweight='bold', color='#1A3A5C')
ax3.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:,.0f}'))
ax3.tick_params(axis='x', rotation=45)
# Annotate peak and trough
peak_idx = df['active_paying_user'].idxmax()
trough_idx = df['active_paying_user'].idxmin()
ax3.annotate(f"Peak\n{df.loc[peak_idx,'active_paying_user']:,}",
             xy=(df.loc[peak_idx,'month'], df.loc[peak_idx,'active_paying_user']),
             xytext=(10, 10), textcoords='offset points', fontsize=8, color=PURPLE)

# ── Chart 4: MoM User Growth % ──
ax4 = fig.add_subplot(4, 2, 4)
colors_user = [RED if x < 0 else PURPLE for x in df_plot['mom_user_growth_pct']]
ax4.bar(df_plot['month'], df_plot['mom_user_growth_pct'], color=colors_user, alpha=0.8, width=20)
ax4.axhline(0, color='black', linewidth=0.8)
ax4.set_title('MoM User Growth %', fontweight='bold', color='#1A3A5C')
ax4.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:.0f}%'))
ax4.tick_params(axis='x', rotation=45)

# ── Chart 5: AOV Trend ──
ax5 = fig.add_subplot(4, 2, 5)
ax5.plot(df['month'], df['aov'], marker='D', color=ORANGE, linewidth=2.5, markersize=5)
ax5.fill_between(df['month'], df['aov'], df['aov'].mean(), alpha=0.1, color=ORANGE)
ax5.axhline(df['aov'].mean(), color=GRAY, linewidth=1, linestyle='--', label=f"Avg: {df['aov'].mean():,.0f}")
ax5.set_title('Average Order Value (AOV)', fontweight='bold', color='#1A3A5C')
ax5.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:,.0f}'))
ax5.legend(fontsize=9)
ax5.tick_params(axis='x', rotation=45)

# ── Chart 6: Growth Driver Divergence ──
ax6 = fig.add_subplot(4, 2, 6)
ax6.plot(df_plot['month'], df_plot['mom_user_growth_pct'], marker='o', color=BLUE,
         linewidth=2, label='User Growth %', markersize=4)
ax6.plot(df_plot['month'], df_plot['mom_aov_growth_pct'], marker='s', color=RED,
         linewidth=2, label='AOV Growth %', markersize=4, linestyle='--')
ax6.axhline(0, color='black', linewidth=0.8)
ax6.set_title('Growth Driver Divergence\n(Users vs AOV)', fontweight='bold', color='#1A3A5C')
ax6.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:.0f}%'))
ax6.legend(fontsize=9)
ax6.tick_params(axis='x', rotation=45)
ax6.fill_between(df_plot['month'],
                  df_plot['mom_user_growth_pct'],
                  df_plot['mom_aov_growth_pct'],
                  alpha=0.05, color=BLUE, label='Gap')

# ── Chart 7: Total Orders ──
ax7 = fig.add_subplot(4, 2, 7)
ax7.bar(df['month'], df['total_orders'], color=GREEN, alpha=0.7, width=20)
ax7.plot(df['month'], df['total_orders'], marker='o', color='#0D6B52', linewidth=1.5, markersize=4)
ax7.set_title('Total Orders per Month', fontweight='bold', color='#1A3A5C')
ax7.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:,.0f}'))
ax7.tick_params(axis='x', rotation=45)

# ── Chart 8: ARPU (extracted) ──
df['arpu'] = df['total_gtv'] / df['active_paying_user']
ax8 = fig.add_subplot(4, 2, 8)
ax8.plot(df['month'], df['arpu'], marker='o', color=ORANGE, linewidth=2.5, markersize=5)
ax8.fill_between(df['month'], df['arpu'], alpha=0.1, color=ORANGE)
ax8.set_title('ARPU (Avg Revenue Per User)', fontweight='bold', color='#1A3A5C')
ax8.yaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f'{x:,.0f}'))
ax8.tick_params(axis='x', rotation=45)
# ARPU trend annotation
arpu_trend = "↑ Improving" if df['arpu'].iloc[-1] > df['arpu'].iloc[0] else "↓ Declining"
arpu_color = GREEN if "Improving" in arpu_trend else RED
ax8.text(0.05, 0.9, arpu_trend, transform=ax8.transAxes,
         fontsize=10, fontweight='bold', color=arpu_color)

plt.tight_layout(rect=[0, 0, 1, 0.97])
plt.savefig('commercial_growth_dashboard.png', dpi=150, bbox_inches='tight',
            facecolor='#FAFAFA')
plt.show()
print("Dashboard saved as commercial_growth_dashboard.png")
```

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
