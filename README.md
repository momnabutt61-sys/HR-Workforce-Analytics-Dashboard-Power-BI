# HR Workforce Analytics Dashboard — Power BI
An end-to-end Power BI project analyzing a HR dataset to answer real workforce planning questions: diversity, geographic distribution, and pay equity across roles and departments. Built with a proper star schema data model, custom DAX measures, and a clean 3-page report design.

## The Business Problem
HR and People Operations teams are regularly asked questions like *"how diverse is our workforce?"*, *"where are we geographically concentrated?"*, and *"is pay consistent across roles and departments?"*, but without a centralized, trustworthy report, these questions usually get answered with slow, manual Excel pulls that go hard immediately and are error-prone to rebuild each time.
This dashboard solves that by turning a raw, messy HR export into a governed, reusable data model with a single source of truth; so, any of these questions can be answered in seconds, self-served, and refreshed automatically as new data arrives, instead of requiring a fresh manual analysis every time leadership asks.

## Dataset
Real-world style HR dataset (1,000 employee records, 17 attributes) sourced from FP20Analytics' Power BI Data Challenge #15. Fields include demographics (gender, ethnicity, age), employment details (hire/exit date, job title, department, business unit), compensation (annual salary, bonus %), and location (country, city, latitude/longitude).

## Tools & Skills Used
- **Power Query (M)** — data cleaning, staging pattern, star schema construction via reference queries and merges
- **Power BI Data Modeling** — star schema design with surrogate keys, role-playing date dimension
- **DAX** — time intelligence, dynamic point-in-time calculations, iterative row-context filtering
- **Power BI Desktop / Report Design** — custom visual layout, conditional formatting, UI/UX-driven dashboard design

## 1. Data Cleaning (Power Query)
Key transformations applied before modeling:

- **Data type correction** across all columns (dates, percentages, whole numbers) — auto-detection isn't reliable and silently breaks DAX aggregations if left unchecked.
- **Text cleaning** (‘Trim’) on all categorical text fields to eliminate invisible whitespace that would otherwise fragment slicers and groupings.
- **Data quality investigation**: discovered that ‘Employee ID’ is **not a unique key** — 172 records share an Employee ID with a completely different person. Identified `Employee New ID` as the true unique identifier and used it throughout the model instead.
- **Derived column**: `Employment Status` (Active / Exited) based on whether `Exit Date` is populated — computed once during ETL rather than repeated inside every DAX measure.
- **Staging query pattern**: the cleaned employee table was frozen as a non-loading staging query (`Stg_Employees`, Enable Load disabled), with the final Fact and Dimension tables built as independent references off that frozen point — avoiding the common Power Query trap where editing an upstream query after other queries reference it silently breaks them.

Every query is fully commented in the M code (see `/power-query` folder) so each Applied Step is self-documenting.

## 2. Data Modeling (Star Schema)
Built a proper star schema instead of one flat table:
| Table | Type | Contents |
|---|---|---|
| `Fact_Employees` | Fact | Employee-level records: salary, bonus %, hire/exit date, age, employment status, and foreign keys to each dimension |
| `Dim_Job` | Dimension | Job Title, Department, Business Unit (245 unique combinations) |
| `Dim_Geography` | Dimension | Country, City, Latitude, Longitude (13 unique locations) |
| `Dim_Gender` | Dimension | Gender, Ethnicity (8 unique combinations) |
| `Dim_Date` | Dimension | Full calendar table (1992–2022), marked as an official Date Table |

**Design decisions worth noting:**
- Each dimension uses a proper **surrogate integer key** (Kimball-style), generated via Power Query's Index Column and resolved onto the fact table through Merge Queries — rather than natural/business keys, which is more resilient to future source data changes.
- `Job Title` alone is not a unique key (e.g., "Director" spans 7 different departments) — this was verified before modeling to avoid building an incorrect relationship. Every dimension's Remove Duplicates step matches on the *full* combination of columns that defines uniqueness, not a single column, after an earlier draft of this model surfaced exactly what goes wrong when that's missed.
- `Dim_Date` has **two relationships** to the fact table: an active one on `Hire Date` and an inactive one on `Exit Date` (activated selectively in DAX via `USERELATIONSHIP` where needed), since a single date table can't be simultaneously active against two different fact-table date columns without ambiguity.

## 3. Key DAX Measures
A dedicated Measures table holds all calculations, separate from any data table — a few examples:

Total Employees = DISTINCTCOUNT ( Fact_Employees[Employee New ID] )

Active Employees = CALCULATE ( [Total Employees], Fact_Employees[Employment Status] = "Active" )

% of Total Employees = 
DIVIDE ( [Total Employees], CALCULATE ( [Total Employees], ALL ( Fact_Employees ) ) )

Avg Annual Salary = AVERAGE ( Fact_Employees[Annual Salary] )

## 4. Report Pages
### Page 1 — Workforce Diversity
Gender, ethnicity, and age-band distribution, plus an ethnicity × gender matrix with conditional formatting to surface intersectional patterns that single-dimension charts can't show.
**Finding:** Black employees represent just 7.4% of the workforce — notably lower than the other three ethnic groups — while gender balance stays close to even (~51/49) within every group.
<img width="1485" height="829" alt="page1-workforce-diversity" src="https://github.com/user-attachments/assets/6ff81c84-efa1-418e-9c08-c024e2bea57c" />

### Page 2 — Geographic Distribution
Interactive bubble map (using true latitude/longitude coordinates, not text-based auto-geocoding, for accuracy) plus a treemap for precise headcount comparison across offices.
**Finding:** 64% of the workforce is US-based across 6 cities, with Seattle as the largest single office (118 employees). Despite the geographic spread, average salary is nearly identical across all three countries (~$112K–114K) — no evidence of location-based pay disparity.
<img width="1483" height="828" alt="page2-geographic-distribution" src="https://github.com/user-attachments/assets/4aaaefe5-9999-46b3-b8c7-5ea2379abc7f" />

### Page 3 — Pay Analysis
Average salary by job title and by department, with the 3-person Board Member/executive tier deliberately isolated from comparative visuals (their $258K average pay is roughly double the next tier and would distort any shared color/axis scale if included).
**Finding:** Excluding the executive tier, department averages span a fairly tight $98K–$129K range. Marketing pays the most among individual-contributor departments ($129K), while IT trails every other department at $98K — a pattern that holds consistently, not just as an isolated department-level average. Job Title tells a complementary story: System Administrator ($203K) unusually outpaces Director ($172K), which would be worth validating with the business in a real engagement rather than assuming it's correct at face value.
<img width="1474" height="832" alt="page3-pay-analysis" src="https://github.com/user-attachments/assets/f20f00c5-6678-480d-a831-3b432a5a681b" />

## What I'd Build Next
With more time, this model is already structured to support: turnover rate trend (using the same `Average Number of Employees` cohort logic as Retention Rate), pay equity analysis controlling for job title and tenure simultaneously, and a bonus-trend page using the existing `Dim_Date` relationship.

## About This Project
Built by **Momna Sajid** as a self-directed portfolio project including data quality investigation, star schema design with surrogate keys, and defensible handling of outliers and partial-year data.
