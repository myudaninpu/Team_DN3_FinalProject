# Education ROI Analysis – SQL Views

This project builds a pipeline of BigQuery views to analyze the **return on investment (ROI) in education** using PISA scores and per-student spending.

---

## Views Created

### 1. `v_spend_clean`
**Purpose:** Normalizes the per-student spend so it’s numeric and consistently named.

```sql
CREATE OR REPLACE VIEW
  `mgmt599-dn3-final-project.edu.v_spend_clean` AS
SELECT
  country,
  CAST(year AS INT64) AS year,
  SAFE_CAST(NY_GDP_PCAP_CD AS FLOAT64) AS spend_per_student_usd
FROM
  `mgmt599-dn3-final-project.edu.data`
WHERE
  NY_GDP_PCAP_CD IS NOT NULL;
```

---

### 2. `v_pisa_roi`
**Purpose:** Joins composite PISA scores to spend and computes ROI = score per dollar.

```sql
CREATE OR REPLACE VIEW
  `mgmt599-dn3-final-project.edu.v_pisa_roi` AS
SELECT
  s.country,
  s.year,
  s.composite_score,
  sp.spend_per_student_usd,
  SAFE_DIVIDE(s.composite_score, sp.spend_per_student_usd) AS score_per_dollar
FROM
  `mgmt599-dn3-final-project.edu.v_pisa_composite_score` AS s
LEFT JOIN
  `mgmt599-dn3-final-project.edu.v_spend_clean` AS sp
ON
  sp.country = s.country
  AND sp.year = s.year;
```

---

### 3. `v_roi_quartiles`
**Purpose:** Buckets countries into ROI quartiles each year (Q1 = highest ROI).

```sql
CREATE OR REPLACE VIEW
  `mgmt599-dn3-final-project.edu.v_roi_quartiles` AS
SELECT
  year,
  country,
  composite_score,
  spend_per_student_usd,
  score_per_dollar,
  NTILE(4) OVER (
    PARTITION BY year
    ORDER BY score_per_dollar DESC
  ) AS roi_quartile
FROM
  `mgmt599-dn3-final-project.edu.v_pisa_roi`
WHERE
  score_per_dollar IS NOT NULL;
```

---

### 4. `v_roi_latest`
**Purpose:** Returns the **most recent ROI per country**, for “current” leaderboards.

```sql
CREATE OR REPLACE VIEW
  `mgmt599-dn3-final-project.edu.v_roi_latest` AS
SELECT
  *
FROM
  `mgmt599-dn3-final-project.edu.v_pisa_roi`
WHERE
  score_per_dollar IS NOT NULL
QUALIFY
  ROW_NUMBER() OVER (
    PARTITION BY country
    ORDER BY year DESC
  ) = 1;
```

---

### 5. `v_pisa_rankings` *(optional helper)*
**Purpose:** Ranks countries by **composite PISA score** (not ROI).

```sql
CREATE OR REPLACE VIEW
  `mgmt599-dn3-final-project.edu.v_pisa_rankings` AS
SELECT
  country,
  year,
  composite_score,
  DENSE_RANK() OVER (
    PARTITION BY year
    ORDER BY composite_score DESC
  ) AS score_rank
FROM
  `mgmt599-dn3-final-project.edu.v_pisa_composite_score`;
```

---

## Pre-Existing Views (Context)
- **`v_pisa_composite_score`**: Canonical composite PISA score per country-year.  
- **`v_pisa_feature_matrix`**: Wide feature table with indicators (student-teacher ratios, digital access, etc.).  

These were already available and serve as inputs to the ROI pipeline.

---

## Summary
The ROI build introduced the following views:
- `v_spend_clean` – Clean spend data  
- `v_pisa_roi` – ROI core table (score per dollar)  
- `v_roi_quartiles` – Quartile buckets  
- `v_roi_latest` – Latest ROI snapshot  

Together, these enable analyses such as:
- Which countries achieve the **highest ROI** in education spending  
- How ROI distribution shifts across quartiles over time  
- Comparing **latest ROI** snapshots versus historical trends  

---
