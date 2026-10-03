# Rohini-Bakery-Business-Analysis
PostgreSQL database architecture and Power BI operational dashboards tracking economic profit margins, owner labor opportunity costs, and a 2-year predictive forecast.
# 🏪 Business Scenario
Rohini is a solo baker who founded her boutique enterprise in 2019, specializing in bespoke, premium custom cakes. To evaluate her operational efficiency, she meticulously recorded her business metrics from 2021 through late 2026, capturing transaction orders, her personal working hours, and monthly operational expenses.
• Data Grain & Privacy: The dataset is structured such that each individual cake represents a separate order record. Customers are uniquely identified via a combination of their name and mobile number. To maintain strict data privacy, all Personally Identifiable Information (PII) has been masked using synthetic customer_id keys, and each order is mapped to a unique order_id.
# 🎯 Core Analytical Objectives
Rohini requires data-driven evidence to resolve three primary strategic questions regarding her enterprise trajectory:
1. Future Trajectory: Based on historical run-rates, what path will the business take over a 2-year forward-looking strategic horizon?
2. Economic Profitability: Is the business genuinely profitable once accounting for operational overhead and the opportunity cost of skilled labor?
3. Business Continuation: What empirical evidence does the data provide to justify continuing or scaling the business operations?

## 🧬 Data Generation & Relational Schema Architecture

### 1. Programmatic Business Simulation
To simulate a complex corporate BI environment, the underlying enterprise dataset was engineered programmatically from scratch using relational SQL scripts. The data model synthetically replicates a high-end boutique business model over a 6-year operational cycle (2021–2026), embedding realistic parameters like varying price points, quantities, and sequential dates.

### 2. Database Schema Blueprint
The relational database structure was deployed within a local PostgreSQL instance. It is engineered using a clean Star Schema design consisting of a Product Dimension table (`cakes`) and a Transactional Fact table (`orders`):

#### A. Product Dimension Table (`cakes`)
This metadata table acts as the master catalog for Rohini's premium product lines.
* **Data Grain:** One row per unique cake category flavor.

| Column Name | Data Type | Key Type | Description / Constraints |
| :--- | :--- | :--- | :--- |
| `cake_id` | `INT` | Primary Key | Unique catalog identifier assigned to each flavor (Values: `101` to `106`). |
| `cake_name` | `VARCHAR(100)` | - | Premium cake classification name. |

* **Master Inventory Catalog:**
  * `101`: Chocolate Cake
  * `102`: Pineapple Cake
  * `103`: Strawberry Cake
  * `104`: Butterscotch Cake
  * `105`: Blackforest Cake
  * `106`: Dry Fruit Plum Cake

#### B. Transactional Fact Table (`orders`)
This table records the granular lifecycle of every individual transaction handled by the bakery.
* **Data Grain:** One row represents an individual cake order.

| Column Name | Data Type | Key Type | Description / Constraints |
| :--- | :--- | :--- | :--- |
| `order_id` | `VARCHAR(50)` | Primary Key | Unique tracking string assigned to every transaction. |
| `customer_id` | `VARCHAR(50)` | - | Masked unique client identifier (Secured synthetic hash; zero PII exposure). |
| `cake_id` | `INT` | Foreign Key | Relational link mapping back to the `cakes` master table. |
| `quantity_kg` | `NUMERIC(4, 2)` | - | Total physical weight of the custom cake order in kilograms. |
| `sold_price` | `NUMERIC(10, 2)` | - | Total gross revenue realized from the sale. |
| `order_date` | `DATE` | - | The exact chronological calendar date the order was logged. |
| `delivery_type` | `VARCHAR(50)` | - | Logistic classification (e.g., Home Delivery, Store Pickup). |

## 📐 Data Transformation Layer: PostgreSQL Views

To optimize query performance and prevent processing lag inside Power BI Desktop, the data transformation workload was entirely pushed back to the database engine. Three specialized relational views were compiled in PostgreSQL to serve as the direct semantic layer for the reporting canvas.

### 1. Monthly Performance Metrics View (`monthly_metrics`)
This view consolidates granular orders data, monthly business overhead expenses, and owner-allocated working hours into a unified chronological summary grouped by month.
```sql
CREATE OR REPLACE VIEW monthly_metrics AS 
WITH orders_mtly AS (
    SELECT
        DATE_TRUNC('month', o.order_date) AS mnth,
        SUM(o.sold_price) AS revenue_mtly
    FROM orders o
    GROUP BY DATE_TRUNC('month', o.order_date)
)
SELECT 
    DATE(om.mnth) AS performance_month,
    COALESCE(om.revenue_mtly, 0) AS revenue_mtly,
    COALESCE(om.revenue_mtly, 0) - COALESCE(e.total_monthly_expenses, 0) AS earnings,
    COALESCE(ow.total_hours_worked, 0) AS total_hours_worked
FROM orders_mtly om
LEFT JOIN expenses e
    ON e.period_start = om.mnth
LEFT JOIN owner_work ow
    ON ow.period_start = om.mnth
ORDER BY performance_month ASC;
```

### 2. Product Categories Monthly Performance View (`cakes_mtly`)
This view handles the multi-table relational join between the transactional fact data and product dimensions, establishing monthly volume tracking and gross revenue streams across the distinct cake classifications.
```sql
CREATE OR REPLACE VIEW cakes_mtly AS
SELECT
	DATE(DATE_TRUNC('month', o.order_date)) AS month,
	c.cake_name,
	COUNT(o.order_id) AS mtly_orders,
	SUM(o.sold_price) AS mtly_revenue
FROM orders o
JOIN cakes c
	ON o.cake_id = c.cake_id
GROUP BY month, c.cake_name;
```

### 3. Progressive Tax Bracket Financial Year View (`yrly_earnings`)
This advanced analytical view shifts operational timelines into a standardized financial year context (starting April 1st) and runs a multi-tier progressive tax calculation model to compute realistic net after-tax corporate earnings.
```sql
CREATE OR REPLACE VIEW yrly_earnings AS
WITH pre_tax AS(
	SELECT 
		EXTRACT(YEAR FROM (performance_month - INTERVAL'3 months')) AS financial_year,
		SUM(earnings) AS pre_tax_earnings
	FROM monthly_metrics
	WHERE performance_month >= '2021-04-01' AND performance_month < '2026-04-01'
	GROUP BY financial_year
)
SELECT
	financial_year,
	ROUND(
		CASE
			WHEN pre_tax_earnings <= 300000 THEN pre_tax_earnings
			WHEN pre_tax_earnings <= 700000 THEN pre_tax_earnings - (pre_tax_earnings - 300000) * 0.05
			WHEN pre_tax_earnings <= 1000000 THEN pre_tax_earnings - (20000 + (pre_tax_earnings - 700000) * 0.10)
			WHEN pre_tax_earnings <= 1200000 THEN pre_tax_earnings - (50000 + (pre_tax_earnings - 1000000) * 0.15)
			WHEN pre_tax_earnings <= 1500000 THEN pre_tax_earnings - (80000 + (pre_tax_earnings - 1200000) * 0.20)
			ELSE pre_tax_earnings - (140000 + (pre_tax_earnings - 1500000) * 0.30)
		END
	, 2) AS earnings
FROM pre_tax;
```
## 📐 Analytics Engine: DAX Modeling & Visual Troubleshooting

### 1. The Custom Opportunity Cost Formula
To answer Rohini's primary question regarding her true economic profitability, a custom DAX measure was formulated. This calculation shifts the perspective from simple bookkeeping to **Economic Profit and Opportunity Cost analysis**. It explicitly subtracts her skilled labor time valuation from the business's realized operational profit:

```dax
True Yearly Earnings = 
SUM(monthly_metrics[earnings]) - (SUM(monthly_metrics[total_hours_worked]) * 1000)
```

* **Strategic Analytical Context:** By establishing a baseline skilled labor cost threshold of **₹1,000 per hour**, the data model calculates whether the enterprise generates a true structural surplus. This provides empirical evidence to determine if the premium pricing model successfully absorbs manual production constraints.

---

### 🛠️ Technical Challenges & Troubleshooting Case Study

#### The Challenge: Filter Context Collapse (The Flat-Line Glitch)
During the creation of the Page 3 Projection trend line, plotting the custom `True Yearly Earnings` measure across the chronological axis resulted in a completely flat, horizontal straight line that repeated a single, identical grand-total number across every plotted interval.

#### Root-Cause Analysis:
The project architecture leverages highly optimized PostgreSQL view layers imported directly into Power BI Desktop without deploying a separate, relational Time-Intelligence Calendar Table. 
* Because the chart's X-axis field was accidentally linked to a mismatching monthly tracking context column (`month` from the `cakes_monthly` table instead of the core `performance_month` transactional scale), the DAX filter context collapsed.
* The visual engine failed to compute row-by-row temporal values, losing track of chronological boundaries and defaulting to the total sum across the whole database scope.

#### Resolution Workflow:
1. **Relational Realignment:** The visual's X-axis field well was scrubbed of disconnected reference points and hard-linked strictly to the native chronological column of the active operational matrix.
2. **Axis Type Reconfiguration:** Within the Visualizations Format panel, the X-axis type property was toggled from **Categorical** (which maps dates as unfilterable text tags) to **Continuous**. 
3. **Outcome:** Activating the native continuous time-series engine immediately restored dynamic row-by-row context filtering. The horizontal line broke apart, snapping into the true moving historical trend path with realistic seasonal ups and downs.

### 2. The Average Order Value Metric
To provide deeper transactional insights into Rohini's customer purchasing patterns across her premium catalog, a secondary DAX measure was implemented on the Page 1 KPI banner to track customer ticket sizes:

```dax
Average Order Value = 
DIVIDE(SUM(monthly_metrics[revenue_mtly]), SUM(cakes_mtly[mtly_orders]), 0)
```

---

## ⏳ Timeline Architecture: Handling the Incomplete FY26 Dataset

A major data cleaning constraint in this project was navigating **the late-2026 partial data lifecycle**. Because the active operational dataset captures real-time monthly records up to **September 2026**, calculating full-year financial run-rates or projecting macro multi-year trajectories using raw aggregates would create severe mathematical distortion (causing a false downward plunge on annual visuals).

To preserve absolute reporting integrity, a dual-layer timeline filtering strategy was deployed across the dashboard canvases:

### 1. Truncating Macro Aggregations to FY25
For high-level financial comparisons, progressive tax structures (`yrly_earnings`), and long-term business continuation decisions, the dataset was strictly truncated to exclude 2026. This limits full annual calculations to the completed historical cycles (**2021, 2022, 2023, 2024, and 2025**). This guarantees that any executive profit baseline evaluated by Rohini represents fully settled financial years.

### 2. Retaining Real-Time Monthly Volume Trends Up to September 2026
While macro annual summaries are locked to 2025, the underlying **chronological monthly charts and product tracking visuals** are allowed to run up to September 2026. 
* **Business Justification:** Omitting late-2026 entirely would blind the business stakeholder to a massive, recent explosive surge in order volumes and market validation. 
* **The Solution:** By explicitly filtering the visual layers, keeping the axis continuous, and adding targeted cards labeled `Revenue 2026 (Jan - Sep)`, the dashboard successfully exposes recent performance breakthroughs without corrupting historical statistical baselines.

## 🎨 Visualization Strategy & Strategic Defense

### 1. Clustered Column Chart vs. 100% Stacked Chart (Page 2)
* **Design Choice:** The multi-year product distribution on Page 2 is deliberately rendered via a **Clustered Column Chart** rather than a 100% Stacked Chart model.
* **Business Justification:** A 100% stacked chart forces every annual column to scale to an identical 100% ceiling height, which completely blinds a business owner to overall volume expansion. Furthermore, low-volume, premium niche products (such as *Strawberry Cake* and seasonal *Dry Fruit Plum Cake*) get crushed into tiny, unreadable slivers at the boundaries of a stacked column. The clustered format preserves their distinct scale, allowing Rohini to verify exactly how many kilograms of these custom flavors sell during specific operational periods.

### 2. The Predictive 2-Year Strategic Horizon (Page 3)
* **Design Choice:** Utilizing a continuous axis baseline derived from fully settled historical cycles (2021–2025), a 2-year automated time-series forecast was generated to project performance across **2026 and 2027**.
* **Interpretation of Bounds:** The chart displays a clear center run-rate accompanied by shaded upper-bound and lower-bound thresholds (**Confidence Intervals**). This explicitly simulates operational risk for the business owner: the center path maps expected trajectory, while the shaded boundaries prove that even in a worst-case downfall or a sudden holiday order surge, operations remain structurally insulated.

---

## 🏁 Core Analytical Conclusions & Executive Decisions

By cross-referencing PostgreSQL transactional view layers with custom opportunity cost modeling, the project successfully delivers data-driven answers to Rohini's primary strategic inquiries:

### 1. What is the trajectory of the business in the upcoming two years?
The 2-year time-series projection demonstrates a clear, stable **upward growth trajectory**. Even when factoring in the conservative lower-bound threshold of the forecasting model, the business maintains a highly stable operational path without plunging into a resource deficit.

### 2. Is the business really profitable?
**Yes, the business is highly viable on an economic profit scale.** When accounting for a standard skilled labor opportunity cost baseline of **₹500 per hour**, the enterprise generated a robust cumulative surplus of **₹1.3 million** across the three fully completed fiscal cycles. However, shifting this constraint to her target **₹1,000 per hour premium tier** tightens her net margins, leaving it to Rohini's executive discretion whether this realized surplus meets her premium lifestyle benchmarks.

### 3. What evidence does the data provide about the continuation of the business?
The empirical evidence strongly supports **business continuation and scaling**:
* **Capacity Management:** The transaction data shows that as order volumes peaked, the strategic onboarding of temporary kitchen support successfully capped Rohini's personal working hours, stabilizing her operational workload.
* **Consistent Momentum:** Her net annual earnings are steadily climbing year-over-year, proving that the premium pricing model successfully absorbs monthly volume fluctuations.
* **Market Validation:** The actual monthly metrics for late 2026 show an explosive, outsized surge in raw order volumes that flies completely above historical statistical expectations. While a secondary analytical sprint is required to isolate the exact driver of this 2026 spike (e.g., specific high-end customer clusters or marketing channels), the current market momentum heavily validates continuing operations.
### 3. Page 1 Macro Charting Decisions: Donut & Clustered Column Layouts
* **Donut Chart Selection (Delivery Type Breakdown):** A Donut Chart was strategically deployed on Page 1 because it isolates a low-cardinality categorical variable (Home Delivery vs. Store Pickup). By displaying only two distinct parts of a whole, it provides an instant visual metric of logistics distribution without creating visual clutter, allowing Rohini to quickly see if her business requires heavy delivery infrastructure.
* **Annual Revenue vs. Net Earnings Clustered Column Chart:** This chart acts as the primary health indicator of the business's scaling efficiency. By placing gross revenue and net post-tax earnings columns directly side-by-side across sequential financial years, the visual instantly exposes the widening or narrowing gap between top-line volume and bottom-line take-home wealth. This maps her operational stress points immediately.
### 3. Page 1 Macro Charting Decisions: Donut & Clustered Column Layouts
* **Donut Chart Selection (Delivery Type Breakdown):** A Donut Chart was strategically deployed on Page 1 because it isolates a low-cardinality categorical variable (Home Delivery vs. Store Pickup). By displaying only two distinct parts of a whole, it provides an instant visual metric of logistics distribution without creating visual clutter, allowing Rohini to quickly see if her business requires heavy delivery infrastructure.
* **Annual Revenue vs. Net Earnings Clustered Column Chart:** This chart acts as the primary health indicator of the business's scaling efficiency. By placing gross revenue and net post-tax earnings columns directly side-by-side across sequential financial years, the visual instantly exposes the widening or narrowing gap between top-line volume and bottom-line take-home wealth. This maps her operational stress points immediately.

## 🎨 Visualization Strategy & Strategic Defense

### 1. Clustered Column Chart vs. 100% Stacked Chart (Page 2)
* **Design Choice:** The multi-year product distribution on Page 2 is deliberately rendered via a **Clustered Column Chart** rather than a 100% Stacked Chart model.
* **Business Justification:** A 100% stacked chart forces every annual column to scale to an identical 100% ceiling height, which completely blinds a business owner to overall volume expansion. Furthermore, low-volume, premium niche products (such as *Strawberry Cake* and seasonal *Dry Fruit Plum Cake*) get crushed into tiny, unreadable slivers at the boundaries of a stacked column. The clustered format preserves their distinct scale, allowing Rohini to verify exactly how many kilograms of these custom flavors sell during specific operational periods.

### 2. The Predictive 2-Year Strategic Horizon (Page 3)
* **Design Choice:** Utilizing a continuous axis baseline derived from fully settled historical cycles (2021–2025), a 2-year automated time-series forecast was generated to project performance across **2026 and 2027**.
* **Interpretation of Bounds:** The chart displays a clear center run-rate accompanied by shaded upper-bound and lower-bound thresholds (**Confidence Intervals**). This explicitly simulates operational risk for the business owner: the center path maps expected trajectory, while the shaded boundaries prove that even in a worst-case downfall or a sudden holiday order surge, operations remain structurally insulated.

---

## 🏁 Core Analytical Conclusions & Executive Decisions

By cross-referencing PostgreSQL transactional view layers with custom opportunity cost modeling, the project successfully delivers data-driven answers to Rohini's primary strategic inquiries:

### 1. What is the trajectory of the business in the upcoming two years?
The 2-year time-series projection demonstrates a clear, stable **upward growth trajectory**. Even when factoring in the conservative lower-bound threshold of the forecasting model, the business maintains a highly stable operational path without plunging into a resource deficit.

### 2. Is the business really profitable?
**Yes, the business is highly viable on an economic profit scale.** When accounting for a standard skilled labor opportunity cost baseline of **₹500 per hour**, the enterprise generated a robust cumulative surplus of **₹1.3 million** across the three fully completed fiscal cycles. However, shifting this constraint to her target **₹1,000 per hour premium tier** tightens her net margins, leaving it to Rohini's executive discretion whether this realized surplus meets her premium lifestyle benchmarks.

### 3. What evidence does the data provide about the continuation of the business?
The empirical evidence strongly supports **business continuation and scaling**:
* **Capacity Management:** The transaction data shows that as order volumes peaked, the strategic onboarding of temporary kitchen support successfully capped Rohini's personal working hours, stabilizing her operational workload.
* **Consistent Momentum:** Her net annual earnings are steadily climbing year-over-year, proving that the premium pricing model successfully absorbs monthly volume fluctuations.
* **Market Validation:** The actual monthly metrics for late 2026 show an explosive, outsized surge in raw order volumes that flies completely above historical statistical expectations. While a secondary analytical sprint is required to isolate the exact driver of this 2026 spike (e.g., specific high-end customer clusters or marketing channels), the current market momentum heavily validates continuing operations.

## ⚠️ Data Limitations & Future Strategic Horizons

### The Missing Variable: Social Media & Marketing Attribution Data
While this business intelligence model delivers highly precise answers based on Rohini's internal transactional, operational hour, and expense datasets, it contains a critical external blind spot: **the lack of marketing and advertising metrics.**

* **The Limitation:** Because social media influence metrics (e.g., Instagram reach, engagement rates, ad spend, follower growth spikes) were not captured or provided by the stakeholder, this analysis cannot mathematically correlate the relationship between product sales and advertising impact.
* **Strategic Risk:** The explosive, outsized surge in orders witnessed in late 2026 cannot be definitively attributed to a specific organic viral post or a targeted paid campaign.
* **Future Analytics Sprint Recommendation:** To transition this dashboard from a *descriptive* tool into an *actionable prescriptive asset*, future operational phases must ingest social media traffic logs. Mapping ad spend data alongside transactional timestamps will allow us to run **Marketing Mix Modeling (MMM)** and calculate precise **Customer Acquisition Cost (CAC)** and **Return on Ad Spend (ROAS)** across her premium cake lines.
