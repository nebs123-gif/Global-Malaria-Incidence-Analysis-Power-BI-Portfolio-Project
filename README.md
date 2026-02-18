Global Malaria Incidence Analysis — Power BI Portfolio Project


 An Interactive Power BI dashboard analyzing 23 years of global malaria incidence data (2000-2022) across 152 geographic entities.


 Project Overview

This end-to-end Business Intelligence project demonstrates data transformation, dimensional modeling, DAX calculations, and interactive visualization capabilities in Power BI. The analysis reveals critical insights into global malaria burden, intervention effectiveness, and geographic distribution patterns.

 Business Problem
Public health organizations, pharmaceutical companies, and international donors need data-driven insights to:
- Identify high-burden regions requiring urgent intervention
- Track progress toward malaria elimination goals
- Allocate resources effectively based on disease burden
- Measure the impact of intervention programs over time

Solution
A two-page interactive Power BI dashboard providing:
- Executive Summary: KPI tracking, trend analysis, and high-burden country identification
- Geographic Analysis: Spatial distribution, regional comparisons, and burden categorization

 Key Insights

 Global Progress
- 91.4 cases per 1,000 population — average global incidence rate
- 31.54% of observations show high burden (>100 cases/1,000)
- 40.7% reduction in global incidence from 2000 to 2022
- Plateau period (2015-2019) followed by renewed decline

 Geographic Distribution
- Sub-Saharan Africa accounts for 96% of global malaria burden
- Burkina Faso highest burden: 584 cases/1,000 population
- Top 10 countries all in West/Central Africa
- 11 countries achieved zero malaria (Egypt, Paraguay, Armenia, etc.)

 Data Coverage
- 3,588 observations across 23 years (2000-2022)
- 107 individual countries tracked
- 71.81% country-level data, 23.49% regional, 4.03% WHO region
- 152 unique geographic entities

 Technical Implementation

 1. Data Source
 File: `malaria_datasetCSV.csv`
- Raw Schema: 14 columns, 3,588 rows
- Source: Global health surveillance data (WHO/partner organizations)
- Coverage: 2000-2022, 152 geographic entities
- Metrics: Incidence rate per 1,000 population with 95% confidence intervals

 2. Data Transformation (Power Query)

 Data Cleaning

// Step 1: Remove 7 redundant columns (50% size reduction)
= Table.RemoveColumns(Source, {
    "IND_ID", "IND_CODE", "IND_UUID", 
    "IND_PER_CODE", "DIM_TIME_TYPE", 
    "DIM_PUBLISH_STATE_CODE", "IND_NAME"
})

// Step 2: Rename to business-friendly names
= Table.RenameColumns(#"Removed Columns", {
    {"DIM_TIME", "Year"},
    {"DIM_GEO_CODE_M49", "GeoCodeM49"},
    {"DIM_GEO_CODE_TYPE", "GeoType"},
    {"GEO_NAME_SHORT", "GeoName"},
    {"RATE_PER_1000_N", "IncidenceRate"},
    {"RATE_PER_1000_NL", "LowerBound"},
    {"RATE_PER_1000_NU", "UpperBound"}
})

// Step 3: Handle null GeoNames (23 rows with M49 code 777)
= Table.ReplaceValue(
    #"Changed Type",
    null,
    "Unknown Region (M49=777)",
    Replacer.ReplaceValue,
    {"GeoName"}
)

Feature Engineering
// Add burden classification column
= Table.AddColumn(
    #"Replaced Nulls",
    "BurdenCategory",
    each if [IncidenceRate] = 0       then "zero"
         else if [IncidenceRate] <= 10  then "low"
         else if [IncidenceRate] <= 50  then "medium"
         else if [IncidenceRate] <= 100 then "high"
         else                               "ver high",
    type text
)

// Add uncertainty metrics
= Table.AddColumn(
    #"Added BurdenCategory",
    "UncertaintyRange",
    each [UpperBound] - [LowerBound],
    type number
)

= Table.AddColumn(
    #"Added UncertaintyRange",
    "UncertaintyPct",
    each if [IncidenceRate] = 0 then null
         else ([UpperBound] - [LowerBound]) / [IncidenceRate] * 100,
    type number
)
```
 3. Data Modeling (Star Schema)

 Model Architecture
              ┌─────────────────┐
              │  dim_Geography  │
              │                 │
              │ • GeoCodeM49 PK │
              │ • GeoName       │
              │ • GeoType       │
              └────────┬────────┘
                       │
                       │ Many-to-One
                       ↓
         ┌─────────────────────────┐
         │    fact_Malaria         │
         │                         │
         │ • Year FK               │
         │ • GeoCodeM49 FK         │
         │ • IncidenceRate         │───────┐
         │ • LowerBound            │       │
         │ • UpperBound            │       │ Many-to-One
         │ • UncertaintyRange      │       ↓
         │ • UncertaintyPct        │  ┌──────────────────────┐
         └─────────┬───────────────┘  │ dim_BurdenCategory   │
                   │                   │                      │
                   │ Many-to-One       │ • BurdenCategory PK  │
                   ↓                   │ • SortOrder          │
          ┌────────────────┐          │ • ColorHex           │
          │   dim_Date     │          └──────────────────────┘
          │                │
          │ • Year PK      │
          │ • Decade       │
          │ • EraLabel     │
          │ • IsRecent5Yr  │
          └────────────────┘

 Table Details

 fact_Malaria (Fact Table)
- Grain: One row per geography per year
- Rows: 3,427 (after filtering)
- Purpose: Central table storing all incidence measurements

dim_Geography (Dimension)
- Rows: 152 unique entities
- Key: GeoCodeM49
- Attributes: Name, Type (Country/Region/WHO/Global)

dim_Date (Dimension)
- Rows: 23 years (2000-2022)
- Key: Year
- Attributes: Decade, Era label, Recent 5-year flag

dim_BurdenCategory (Dimension)
- Rows: 5 categories
- Key: BurdenCategory
- Purpose: Controls sort order and conditional formatting colors

4. DAX Measures

 Core Aggregations
dax
// Average incidence rate in current filter context
Avg Incidence Rate = 
AVERAGE(fact_Malaria[IncidenceRate])

// Count of observations
Total Records = 
COUNTROWS(fact_Malaria)

// Percentage of high-burden records
% High Burden Records = 
DIVIDE(
    CALCULATE(
        COUNTROWS(fact_Malaria),
        fact_Malaria[IncidenceRate] > 100
    ),
    [Total Records],
    0
)

// Countries at zero malaria
Countries at Zero = 
CALCULATE(
    DISTINCTCOUNT(fact_Malaria[GeoCodeM49]),
    fact_Malaria[IncidenceRate] = 0,
    dim_Geography[GeoType] = "COUNTRY"
)


 Time Intelligence
dax
// Year-over-year change
YoY Change = 
VAR CurrentRate = [Avg Incidence Rate]
VAR PriorRate = 
    CALCULATE(
        [Avg Incidence Rate],
        DATEADD(dim_Date[Year], -1, YEAR)
    )
RETURN
    IF(
        ISBLANK(PriorRate),
        BLANK(),
        CurrentRate - PriorRate
    )

// Year-over-year percentage change
YoY % Change = 
DIVIDE(
    [YoY Change],
    ABS(CALCULATE(
        [Avg Incidence Rate],
        DATEADD(dim_Date[Year], -1, YEAR)
    )),
    BLANK()
)

// 5-year rolling average (smooths volatility)
5-Year Rolling Avg = 
CALCULATE(
    [Avg Incidence Rate],
    DATESINPERIOD(
        dim_Date[Year],
        LASTDATE(dim_Date[Year]),
        -5,
        YEAR
    )
)

// Change from 2000 baseline
Change Since 2000 = 
VAR Baseline = 
    CALCULATE(
        [Avg Incidence Rate],
        dim_Date[Year] = 2000,
        ALL(dim_Date)
    )
VAR Current = [Avg Incidence Rate]
RETURN
    DIVIDE(Current - Baseline, Baseline, BLANK())

 Uncertainty Metrics
dax
// Average estimation uncertainty (excludes zero-incidence)
Avg Uncertainty % = 
AVERAGEX(
    FILTER(
        fact_Malaria,
        fact_Malaria[IncidenceRate] > 0
    ),
    fact_Malaria[UncertaintyPct]
)

// Confidence interval width
Confidence Band Width = 
AVERAGE(fact_Malaria[UncertaintyRange])
```

5. Dashboard Design

 Page 1: Executive Summary
Purpose: High-level KPIs and trend analysis

| Visual Type | Fields | Purpose |
|------------|--------|---------|
| KPI Card | Avg Incidence Rate | Primary metric (91.4) |
| KPI Card | Countries at Zero | Success metric (11) |
| KPI Card | % High Burden Records | Risk indicator (31.54%) |
| Line Chart | Year → Avg Incidence Rate | Temporal trend (2000-2022) |
| Bar Chart | GeoName → Incidence Rate (Top 11) | Highest burden countries |
| Table | Zero-incidence countries | Elimination success stories |
| Clustered Bar | IncidenceRate + Bounds | Uncertainty visualization |
| Donut Chart | GeoType breakdown | Data composition |
| Slicers | Year range, Burden category | Interactive filtering |

Design Decisions:
- Large KPI cards for immediate impact
- Line chart shows clear downward trend with plateau period
- Bar chart ordered by burden (descending) with data labels
- Confidence intervals visualize data reliability
- White/rounded design for professional appearance

 Page 2: Geographical Analysis
Purpose: Spatial patterns and regional comparisons

| Visual Type | Fields | Purpose |
|------------|--------|---------|
| Filled Map | GeoName → Incidence Rate | Global spatial distribution |
| Bar Chart | GeoType → Avg Incidence | Entity-level comparison |
| Bar Chart | BurdenCategory counts | Distribution of burden levels |
| Line Chart | Year × Burden × GeoType | Multi-dimensional trends |

Design Decisions:
- Map visual immediately shows Africa concentration
- Consistent color scheme across pages
- Sorted burden categories (low → very high)
- Multi-line chart shows category evolution over time

Conditional Formatting Rules

| Element | Rule | Purpose |
|---------|------|---------|
| Burden Category | Field value → custom colors | Visual consistency |
| Bar colors | Diverging red scale | Intensity indication |
| Map saturation | Gradient (white → dark red) | Burden density |
| YoY Change | Red (increase) / Green (decrease) | Direction signaling |

6. Interactivity Features

 Cross-Filtering
- Clicking any visual filters all related visuals on the page
- Year slicer synchronized across both pages (Sync Slicers enabled)
- Burden category filter excludes/includes burden levels dynamically

 Drill-Through Capability
- Users can right-click any country → Drill-through for detailed view
- Maintains filter context from source page

 Mobile Layout
- Responsive design principles applied
- Portrait orientation optimized for tablets
- KPI cards prioritized in mobile view


 Business Value & Impact

 Decision Support
1. Resource Allocation: Identifies 15 countries accounting for bulk of global burden
2. Program Evaluation: Tracks 40.7% reduction over 22 years
3. Target Setting: 11 countries demonstrate elimination is achievable
4. Risk Assessment: 31.54% high-burden rate quantifies intervention urgency

 Stakeholder Benefits

**Public Health Organizations**
- Prioritize interventions in West/Central Africa
- Replicate success models from zero-malaria countries
- Monitor stagnation periods to prevent backsliding

Pharmaceutical Companies
- Market sizing: 584 cases/1,000 in highest-burden countries
- Product positioning: Prevention vs. elimination tools by region
- R&D priorities: Address 52% average uncertainty in estimates

International Donors
- Evidence-based funding allocation
- Progress monitoring against elimination goals
- ROI measurement for intervention programs

Government Agencies
- National strategy development
- Cross-border collaboration (regional patterns)
- Surveillance system improvement (data quality)


Skills Demonstrated

   Technical Skills
- ✅ Power Query (M): Complex transformations, null handling, feature engineering
- ✅ Data Modeling: Star schema design, relationship management, cardinality optimization
- ✅ DAX: Time intelligence, iterator functions, context transition, measure dependencies
- ✅ Data Visualization: Multi-page dashboards, conditional formatting, color theory
- ✅ UX Design: Interactive filtering, drill-through, mobile responsiveness

  Business Skills
- ✅ Domain Knowledge: Public health metrics, epidemiological concepts, WHO classifications
- ✅ Analytical Thinking: Identifying patterns, correlating trends, root cause analysis
- ✅ Storytelling: Two-page narrative (overview → detail), clear KPI hierarchy
- ✅ Stakeholder Focus: Tailored insights for diverse audiences (NGOs, pharma, governments)

    Best Practices Applied
- ✅ Performance Optimization: Removed 50% of columns, star schema reduces redundancy
- ✅ Maintainability: Centralized measures table, descriptive naming conventions
- ✅ Scalability: Model supports easy addition of new years/geographies
- ✅ Documentation: In-line comments in DAX, clear transformation steps
- ✅ Data Quality: Null handling, explicit type conversion, uncertainty quantification


 Key Learnings & Challenges

  Challenges Overcome
1. Null Geography Names: 23 rows with M49 code 777 had no labels
   - Solution: Replaced with descriptive placeholder, investigated UN M49 codes
   
2. Uncertainty Handling: Division by zero when calculating uncertainty percentage
   - Solution: Null guard in Power Query (returns null, not 0)

3. Time Intelligence with Years: DATEADD requires date table, not integer year
   - Solution: Created proper dim_Date table with year as date type

4. Burden Category Sorting: Alphabetical sort (high → low → medium → ...) was illogical
   - Solution: Created dim_BurdenCategory with SortOrder column

Best Practices Learned
- Always create a separate measures table (_Measures) — don't store measures in data tables
- Use DIVIDE() instead of "/" operator — prevents #ERROR from division by zero
- Star schema outperforms flat tables for DAX performance (3x faster in testing)
- Explicit data typing in Power Query prevents silent failures
- Descriptive names > technical names (GeoName vs DIM_GEO_CODE_SHORT)

---

 Future Enhancements

 Phase 2 Improvements
- [ ] Predictive Analytics: Prophet/ARIMA forecasting for 2023-2030 projections
- [ ] What-If Analysis: Parameter-based scenario modeling (e.g., "What if interventions increase 20%?")
- [ ] Drill-Down Pages: Country-specific deep dives with intervention timelines
- [ ] Anomaly Detection: Flag unusual spikes/drops in incidence rates
- [ ] Correlation Analysis: Link with intervention funding data, GDP, healthcare spending

Data Enhancement
- [ ] Mortality Data: Add malaria death rates for complete burden assessment
- [ ] Intervention Data: Bed net distribution, ACT availability, funding levels
- [ ] Climate Data: Temperature, rainfall correlation with incidence
- [ ] Population Demographics: Age-stratified analysis (children under 5)

Technical Upgrades
- [ ] Power BI Service: Publish to workspace with Row-Level Security (RLS) for regional users
- [ ] Automated Refresh: Direct Query or scheduled refresh from SQL database
- [ ] R/Python Visuals: Advanced statistical charts (regression, clustering)
- [ ] Power Automate: Alert stakeholders when high-burden threshold exceeded


Open to
- BI/Analytics roles in global health, public health, or life sciences
- Collaboration on malaria/infectious disease data projects
- Consulting on Power BI implementations
- Guest speaking on health data visualization


License & Attribution

Data Source: Global Malaria Dataset (WHO/Partner Organizations)  
Tool: Microsoft Power BI Desktop (Version 2.XX)  
License: MIT License (code/transformations), Data subject to source terms

### Citation
If using this work, please cite:
[michael iyede]. (2024). Global Malaria Incidence Analysis - Power BI Dashboard. 
GitHub Repository: nebs123-gif


 Acknowledgments

- World Health Organization — for malaria surveillance data
- Power BI Community — for DAX patterns and best practices

 References & Resources

 Technical Resources
- [DAX Patterns](https://www.daxpatterns.com/) — DAX calculation reference
- [SQLBI](https://www.sqlbi.com/) — Star schema & modeling best practices
- [Power BI Docs](https://docs.microsoft.com/en-us/power-bi/) — Official documentation

Domain Knowledge
- [WHO Malaria Report](https://www.who.int/teams/global-malaria-programme/reports/world-malaria-report-2023) — Annual statistics
- [Roll Back Malaria](https://rollbackmalaria.com/) — Global partnership
- [Our World in Data - Malaria](https://ourworldindata.org/malaria) — Data journalism



