# Churn Prediction:

## Types of Churns:
- Voluntary Churn (Active Cancellation)
- Involuntary Churn
- Passive Churn
- Seasonal Churn
- Downgrade Churn
- Forced Churn
- Early Tenure Churn

## Voluntary Churn:
Where the customer consciously cancels the subscription

### Reasons:
- Not using car wash much
- Too expensive
- Moved to different city
- Bad Service experience
- Seasonal Usage drop
- Sold vehicle

### Signals:
- days_since_last_visit
- Declining wash frequency
- No promo engagement
- Auto-renew often turned OFF before cancel

## Involuntary Churn:
Customer didn't cancel intentionally, subscription stopped due to failed payment (cases where the customer may not even notice)

### How this may look:
System attempts charge -> Payment failed -> Retry 1 -> Retry 2 -> Retry 3 -> Subscription suspended

### Reason:
- Card expired
- Insufficient funds
- Bank decline
- Fraud block
- Forgot to renew, if the Auto-Pay = FALSE

> [!WARNING]
> This is dangerous - Customers may still want the service (Easy to winback)

### Signals:
- payment_failed_last_30d
- retry count
- days_since_last_successful_payment
- card expiry flag

## Passive Churn (Usage drop without cancellation):
Customer stops using the service but technically their subscription is still active

### How this may look:
Auto-renew = TRUE -> Still paying monthly -> Has not washed car in the last 45+ days

> [!WARNING]
> High future churn risk<br>
> They may cancel soon<br>
> They may dispute charge

## Seasonal Churn (Temporary):
This is cyclical churn.
- Winter heavy usage
- Summer minimal usage
- Cancels in low season
- Re-subscribes in winter

> [!WARNING]
> Need to cautious about this churn, this may lead to over-predict churn in summer and under-predict in the winter

> [!IMPORTANT]
> Since JAX is based in Michigan, we need to consider this

## Downgrade Churn:
Not full churn, but revenue churn

### How this may look:
- Moves from $39.99 plan → $25.99 plan

Revenue drop but customer retained, For business this is still partial churn

## Forced Churn:
Less common but important
- Fraud detected
- Abuse of unlimited plan
- Excessive daily washes
- Policy violation

## Early Tenure Churn:
Customers who churn within first 1–2 months.
- Wrong expectations
- Didn’t see value
- Poor onboarding

This type behaves differently from long-term churners.

### Features:
- tenure_months
- first_month_usage



Auto-renew OFF is often a strong churn predictor

```
If:
auto_renew = FALSE
AND low wash frequency
AND tenure < 3 months
```
-> High churn probability.

## Business Insight
In many subscription companies:
20–40% churn = involuntary (payment)
60–80% churn = voluntary

If you reduce payment churn by:
- Updating expired cards
- Smart retries
- SMS reminders

You can reduce total churn significantly without changing product


## Features for Jax churn model

### Usage / DRD (visit behavior)
- washes_last_7d / 30d / 90d
- days_since_last_wash
- usage_trend (washes_last_30d vs previous_30d)
- max_gap_days_last_90d
- preferred_location, location_switch_count
- weekday/weekend ratio
- time-of-day habits (optional)

### Subscription state
- plan_tier (basic / premium)
- monthly_price
- tenure_months
- plan_changes_count (upgrade/downgrade)
- auto_renew flag
- pause/suspend status (if exists)

### Payments & billing (involuntary churn engine)
- payment_failures_last_30d / 90d
- retry_count_last_bill
- days_since_last_successful_payment
- “in_dunning” flag (if you have dunning steps)
- card_expiring_soon flag (if available)
- outstanding_balance / chargeback flags (if applicable)

### Customer support / experience
- complaints_last_90d
- refund_count
- service_quality flags, rewash requests

### Seasonality / external context
- month, week_of_year
- Seasonal flags (is_winter, is_spring, is_summer)
- Instead of just washes_last_30d, we can also use ```relative_usage = washes_last_30d / avg_washes_for_all_customers_in_same_month```
- consecutive_payment_failures
- first_time_failure_flag
- retry_count_trend

### Features to capture the time-series:
- usage_momentum = (washes_last_30d - washes_prev_30d) - (washes_prev_30d - washes_prev_60d)

## Prediction:
- **Will customer churn in next N days (common N = 30) [binary - 0 or 1]**
- What type of churn (0 - No Churn, 1 - Voluntary Churn, 2 - Involuntary Churn) - Heavy data analysis and pre-processing would be needed to prepare the training data
- Survival Analysis (Spotify uses this) [doc](https://research.atspotify.com/2022/11/survival-analysis-meets-reinforcement-learning)
