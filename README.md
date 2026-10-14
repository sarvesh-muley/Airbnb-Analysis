# Airbnb-Analysis

# Airbnb Market Analysis & Executive Decision Support Dashboard

## Executive Summary
This project provides an end-to-end Business Intelligence solution developed in Microsoft Power BI to evaluate Airbnb market performance, pricing anomalies, supply-demand dynamics, host quality tiers, and inventory availability. Built on a dataset covering multi-city listings and historical reviews, the report translates raw operations data into five strategic decision areas:

1. **Listing Performance:** Identifying top-performing asset types and dimensions driving guest satisfaction.
2. **Pricing Strategy:** Detecting overpriced and underpriced listings relative to peer medians.
3. **Location & Market Demand:** Mapping geographic demand hubs and identifying underserved sub-markets.
4. **Host Performance:** Measuring the business impact of Superhost status and host response velocity.
5. **Availability & Occupancy Potential:** Auditing dormant listings and evaluating the impact of stay restrictions.

---

## Data Architecture & Data Model

The data model is structured using an analytical Star Schema to optimize DAX query performance and simplify filter propagation:

* **Fact Table (`Reviews`):** Contains historical guest review events (`listing_id`, `date`, `review_id`).
* **Dimension Table (`Listings`):** Contains property metadata, pricing, location attributes, stay requirements, and host performance profiles (`listing_id`, `host_id`, `neighbourhood`, `room_type`, `price`, `minimum_nights`, etc.).
* **Dimension Table (`Dim_Date`):** Dedicated calendar table marking contiguous dates, quarters, years, year-months, and day-of-week attributes.
* **Calculation Group / Metric Repository (`_Measures`):** A dedicated, disconnected repository housing all explicit business DAX metrics.

### Model Relationships
* `Listings[listing_id]` (1) $\rightarrow$ `Reviews[listing_id]` ($\infty$) | Single directional filter
* `Dim_Date[Date]` (1) $\rightarrow$ `Reviews[date]` ($\infty$) | Single directional filter

---

## Power Query Data Transformation & Cleaning Pipeline

The raw CSV extracts presented several data quality challenges, including character encoding anomalies (mojibake), non-standard boolean values, and missing values. The following cleaning transformations were applied in Power Query prior to model loading:

### 1. Character Encoding Standardization
* **Issue:** Property names containing multi-byte UTF-8 characters (e.g., Thai, Chinese, accent marks, smart quotes) exhibited mojibake when ingested using default code page decoders.
* **Solution:** Explicitly configured the extraction step to use Code Page `65001: Unicode (UTF-8)`. In reporting visuals, listing identification was further standardized to use `listing_id` alongside clean location descriptors to ensure tabular uniformity and eliminate formatting breaks.

### 2. Control Character & Text Normalization
* **Transformation:** Embedded carriage returns (`#(cr)`) and line feeds (`#(lf)`) inside property descriptions were sanitized using Power Query's `Text.Clean` and `Text.Trim` functions, supplemented by explicit value replacements to eliminate multi-line row stretching in report visuals.

### 3. Boolean Standardization
* **Issue:** Flag columns stored booleans as string indicators (`"t"` and `"f"`), which prevented native aggregation and boolean evaluation.
* **Transformation:** Applied two sequential value replacement operations on `host_is_superhost`, `host_identity_verified`, and `instant_bookable`:
  * Replaced `"t"` with `"true"`
  * Replaced `"f"` with `"false"`
  * Converted explicit data types to `True/False (Logical)`.

### 4. Data Type Corrections & Column Casting
* Stripped currency and text symbols from pricing and response metrics.
* Cast `price`, `review_scores_rating`, and sub-category scores to Decimal Numbers.
* Cast `minimum_nights`, `accommodates`, and `review_id` to Whole Numbers.
* Cast `host_since` and `first_review` to standard Date formats.

---

## Explicit DAX Measures & Business Rationale

To deliver clean, dynamic calculations across synced slicers (`city`, `room_type`, `Year`), all business logic was written as explicit DAX measures in the `_Measures` table.

### 1. Volume & Scale Measures

#### `Total Listings`
```dax
Total Listings = COUNTROWS(Listings)
