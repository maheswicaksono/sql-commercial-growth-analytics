# sql-commercial-growth-analytics
MoM commercial growth analysis evaluating GTV, net revenue, active users, and AOV using SQL window functions &amp; Python visualization.

# Fintech Business Performance Analysis

Analysis of fintech transaction data (19 months) to understand growth drivers: user acquisition vs. monetization per user.

### Metrics Defined

1. **GTV (Gross Transaction Value)**
   - Definition: `SUM(amount)` for all successful transactions
   - Unit: Currency (IDR assumed, but currency-agnostic in calc)

2. **Net Revenue**
   - Definition: `GTV × 2.5%` (assumed flat take rate)
   - Assumption: Justification in METHODOLOGY.md

3. **Active Paying Users**
   - Definition: `COUNT(DISTINCT user_id)` per month with at least 1 successful transaction
   - Excludes: Cancelled, pending, failed transactions

4. **Average Order Value (AOV)**
   - Definition: `GTV / Total Orders`
   - Interpretation: Basket size per transaction

5. **MoM Growth %**
   - Formula: `(Current - Previous) / Previous × 100%`
   - Handling: NULLIF to prevent division-by-zero on first month

## Key Findings

- **Volume-driven growth**: User acquisition is the primary growth lever; AOV is flat
- **Decelerating trend**: Growth rate cooling from 188% (Feb 2025) to -13% (Jul 2026)
- **Jul 2026 contraction**: All metrics negative — warrants investigation (seasonal vs. structural)
- **Take rate assumption**: Net revenue modeled as flat 2.5% of GTV

PURPOSE:
  Calculate key business metrics (GTV, Revenue, Users, AOV) and their 
  month-over-month growth rates for trend analysis.

KEY ASSUMPTIONS:
  - Take rate: 2.5% (flat, applies equally to all transactions)
  - Filter: Only successful transactions (status = 'success')
  - User counting: Distinct per month (repeat users counted once per month)
  - Rounding: Precision preserved until final output layer

METHODOLOGY:
  1. raw_monthly    : Base aggregation (sum, count by calendar month)
  2. calculated_monthly : Derive ratios + LAG for prior month
  3. final SELECT   : Round for display, calculate MoM growth %

OUTPUTS:
  - month: Calendar month (YYYY-MM format)
  - total_gtv: Gross Transaction Value (currency)
  - total_net_revenue: GTV × 2.5% take rate (currency)
  - aov: Average Order Value per transaction
  - active_paying_user: Distinct users with successful txns
  - total_orders: Count of successful transactions
  - mom_gtv_growth_pct: Month-over-month growth %
  - mom_net_revenue_growth_pct: MoM revenue growth %
  - mom_user_growth_pct: MoM active user growth %
  - mom_aov_growth_pct: MoM average order value growth %
    
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

## Queries

df_result = q("""
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

""")
display(df_result.head(20))


## What I'd Do Next

If extending this analysis:
1. Cohort retention table (are 2025 users stickier than 2026?)
2. Segment by product category (which products drive growth?)
3. Is Jul contraction seasonal? (need 2-3 years of data to confirm)

---

**Status**: ✅ Complete  
**Dataset**: 2025-01 to 2026-07 (synthetic)
