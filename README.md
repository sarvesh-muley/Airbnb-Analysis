# Airbnb Market Analysis & Executive Decision Support Dashboard

![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Measures-0078D4)
![Power Query](https://img.shields.io/badge/Power%20Query-ETL-217346)

An end-to-end Business Intelligence solution built in **Microsoft Power BI** to evaluate Airbnb market performance, pricing anomalies, supply-demand dynamics, host quality tiers, and inventory availability.

## Table of Contents

- [Project Overview](#project-overview)
- [Data Architecture](#data-architecture)
- [Data Cleaning Pipeline](#data-cleaning-pipeline)
- [DAX Measures](#dax-measures)
- [Report Structure](#report-structure)
- [Key Findings](#key-findings)

---

## Project Overview

The project uses a multi-city dataset of listings and historical reviews, and turns raw operational data into five strategic decision areas:

| # | Decision Area | Objective |
|---|---------------|-----------|
| 1 | **Listing Performance** | Identify top-performing property configurations and the factors driving guest satisfaction |
| 2 | **Pricing Strategy** | Detect overpriced and underpriced listings relative to local peer medians |
| 3 | **Location & Market Demand** | Map demand hubs and identify underserved sub-markets |
| 4 | **Host Performance** | Measure the impact of Superhost status and response speed |
| 5 | **Availability & Occupancy** | Audit dormant listings and assess the effect of minimum-stay rules |

---

## Data Architecture

The model follows a **Star Schema** to optimise DAX performance and keep filter propagation predictable.

| Table | Type | Description |
|-------|------|-------------|
| `Reviews` | Fact | Historical guest review events (`listing_id`, `date`, `review_id`) |
| `Listings` | Dimension | Property metadata, pricing, location, stay rules, host profile (`listing_id`, `host_id`, `neighbourhood`, `room_type`, `price`, `minimum_nights`) |
| `Dim_Date` | Dimension | Calendar table with dates, quarters, years, year-months, day-of-week |
| `_Measures` | Measure repository | Disconnected table holding all explicit DAX measures |

### Relationships

- `Listings[listing_id]` (1) → `Reviews[listing_id]` (*) — single direction
- `Dim_Date[Date]` (1) → `Reviews[date]` (*) — single direction

---

## Data Cleaning Pipeline

Raw data quality issues were resolved in **Power Query** before loading.



### 1. Boolean Normalisation

- **Issue:** Flag columns stored `"t"` / `"f"` strings, blocking conditional logic in DAX.
- **Fix:** On `host_is_superhost`, `host_identity_verified`, and `instant_bookable`:
  1. Replaced `"t"` → `"true"`
  2. Replaced `"f"` → `"false"`
  3. Cast to **True/False** type

### 2. Data Type Corrections

| Columns | Type |
|---------|------|
| `price`, `review_scores_rating`, sub-category scores | Decimal Number |
| `minimum_nights`, `accommodates`, `review_id` | Whole Number |
| `host_since`, `first_review` | Date |

Currency symbols and text formatting were stripped from price and rate fields before casting.

---

## DAX Measures

All business logic lives in the `_Measures` table and responds to the synced slicers (`city`, `room_type`, `Year`).

### 1. Volume & Scale

```dax
Total Listings = COUNTROWS(Listings)
```
Primary denominator for market supply sizing.

```dax
Total Reviews = COUNTROWS(Reviews)
```
Proxy for completed bookings and transaction volume.

```dax
Total Hosts = DISTINCTCOUNT(Listings[host_id])
```
Separates operator scale from property count to reveal multi-unit concentration.

### 2. Pricing & Market Benchmarks

```dax
Avg Price = AVERAGE(Listings[price])
```
Mean nightly price across filtered properties.

```dax
Median Price = MEDIAN(Listings[price])
```
Resistant to ultra-luxury outliers; reflects the typical rate paid.

```dax
Peer Median Price =
CALCULATE(
    MEDIAN(Listings[price]),
    ALLEXCEPT(Listings, Listings[neighbourhood], Listings[room_type])
)
```
Local benchmark for a property within its neighbourhood and room type.

```dax
Price Index =
DIVIDE(
    [Avg Price],
    CALCULATE([Avg Price], ALLSELECTED(Listings[neighbourhood]))
)
```
Normalises price against the market baseline. `1.00` = parity; above `1.00` = premium.

### 3. Pricing Exceptions & Opportunities

```dax
Listings to Review Pricing =
CALCULATE(
    [Total Listings],
    Listings[price] > 1.5 * Listings[Peer Median Price],
    Listings[review_scores_rating] < 90,
    NOT(ISBLANK(Listings[review_scores_rating]))
)
```
Flags listings priced above 150% of peer median with ratings below 90.

```dax
Listings to Raise Price =
CALCULATE(
    [Total Listings],
    Listings[price] < 0.7 * Listings[Peer Median Price],
    Listings[review_scores_rating] >= 95
)
```
Flags top-rated listings (≥ 95) priced below 70% of peer median, which indicates uncaptured revenue.

### 4. Quality & Host Performance

```dax
Avg Rating = AVERAGE(Listings[review_scores_rating])
```
Cumulative guest satisfaction on a 0–100 scale.

```dax
Superhost % =
DIVIDE(
    CALCULATE(DISTINCTCOUNT(Listings[host_id]), Listings[host_is_superhost] = TRUE()),
    [Total Hosts],
    0
)
```
Share of hosts with Superhost status.

```dax
Avg Response Rate = AVERAGE(Listings[host_response_rate])
```
Indicator of host communication quality.

```dax
Hosts Needing Improvement =
CALCULATE(
    [Total Hosts],
    Listings[review_scores_rating] < 90 ||
    Listings[host_response_rate] < 0.80 ||
    Listings[Dormant %] > 0.50
)
```
Hosts failing minimum thresholds on rating, response rate, or dormancy.


### 5. Availability, Recency & Stay Restrictions

```dax
Active % =
DIVIDE(
    CALCULATE([Total Listings], Listings[Days Since Last Review] <= 180),
    [Total Listings],
    0
)
```
Share of listings reviewed in the last 6 months.

```dax
Dormant % =
DIVIDE(
    CALCULATE(
        [Total Listings],
        Listings[Days Since Last Review] > 365 || ISBLANK(Listings[Days Since Last Review])
    ),
    [Total Listings],
    0
)
```
Share of listings with no review in 12+ months or none at all.

```dax
Long Min-Stay % =
DIVIDE(
    CALCULATE([Total Listings], Listings[minimum_nights] >= 30),
    [Total Listings],
    0
)
```
Prevalence of 30+ night minimum stays, which suppress booking velocity.

```dax
Reviews YoY % =
VAR CurrentYearReviews = [Total Reviews]
VAR PreviousYearReviews = CALCULATE([Total Reviews], SAMEPERIODLASTYEAR(Dim_Date[Date]))
RETURN
    DIVIDE(CurrentYearReviews - PreviousYearReviews, PreviousYearReviews, 0)
```
Year-over-year demand trajectory.

---

## Report Structure

| Page | Title | Content |
|------|-------|---------|
| 1 | **Market Overview** | Executive KPIs, review trends, inventory by room type, geographic density |
| 2 | **Listing Performance** | Room-type matrix, price vs. rating, sub-scores (cleanliness, accuracy, communication), top listings |
| 3 | **Pricing Strategy** | Median price by neighbourhood, price tiers, price index heatmap, pricing exception table |
| 4 | **Location & Market Demand** | Supply vs. demand matrix, top-demand neighbourhoods, annual review growth |
| 5 | **Host Performance** | Superhost vs. standard hosts, response-time impact, host coaching roster |
| 6 | **Availability & Occupancy** | Active / Slowing / Dormant split, minimum-night distribution, inactive inventory audit |
| 7 | **Executive Insights** | Decision matrix with recommendations across all five areas |

---

## Key Findings

- **Dormant inventory cleanup:** Over 65% of listings show no guest activity in the trailing 12 months. Automated delisting or search deprioritisation would improve search relevance and conversion.
- **Revenue from underpriced assets:** Many top-rated listings (rating ≥ 95) are priced over 30% below peer medians. Nudging nightly rates up 10–15% should add revenue without hurting booking pace.
- **Overpriced risk:** Premium-priced listings rated below 90 erode guest trust. Price caps or warnings can steer rates toward peer medians until quality improves.
- **Asset class allocation:** "Entire place" listings lead on average daily rate, composite score, and demand. Onboarding should prioritise full residential units over shared spaces.
- **Service-level standards:** Hosts responding within one hour earn consistently higher ratings than those responding after 24 hours. Response-time SLAs should be part of host coaching.

---

## Tech Stack

- Microsoft Power BI Desktop
- Power Query (M)
- DAX
