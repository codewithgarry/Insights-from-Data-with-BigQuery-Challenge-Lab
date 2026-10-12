# Insights from Data with BigQuery: Challenge Lab (GSP787)

> 📦 **Archived.** This lab is now maintained in **[google-cloud-skills-boost-notes](https://github.com/codewithgarry/google-cloud-skills-boost-notes/tree/main/data-bigquery/gsp787-insights-from-data-with-bigquery)** with my other Google Cloud lab notes. This repository is read-only.

![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?logo=googlecloud&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-GSP787-34A853)
![BigQuery](https://img.shields.io/badge/BigQuery-669DF6?logo=googlebigquery&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?logo=mysql&logoColor=white)

Reference solution for the Google Cloud Skills Boost challenge lab **Insights from Data with BigQuery** (GSP787).

## 📖 Overview

You answer ten analytical questions about the early COVID-19 pandemic with standard SQL. The queries use aggregation, CTEs, window functions (`LAG`, `LEAD`) and growth-rate math.

**Dataset:** `bigquery-public-data.covid19_open_data.covid19_open_data`

## 🧰 Skills demonstrated

Aggregations · Common table expressions · Window functions · Date arithmetic · Ratio and growth-rate calculations

> Dates and thresholds change between lab versions. Replace them with the values in your instructions.

## 🚀 Solution

### Query 1: Total confirmed cases worldwide

```sql
SELECT SUM(cumulative_confirmed) AS total_cases_worldwide
FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE date = '2020-04-15';
```

### Query 2: US states with more than 100 deaths

```sql
WITH deaths_by_state AS (
  SELECT subregion1_name AS state, SUM(cumulative_deceased) AS death_count
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE country_name = 'United States of America'
    AND date = '2020-04-10'
    AND subregion1_name IS NOT NULL
  GROUP BY subregion1_name
)
SELECT COUNT(*) AS count_of_states
FROM deaths_by_state
WHERE death_count > 100;
```

### Query 3: Hotspot states with more than 1,000 confirmed cases

```sql
SELECT *
FROM (
  SELECT subregion1_name AS state, SUM(cumulative_confirmed) AS total_confirmed_cases
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE country_code = 'US'
    AND date = '2020-04-10'
    AND subregion1_name IS NOT NULL
  GROUP BY subregion1_name
)
WHERE total_confirmed_cases > 1000
ORDER BY total_confirmed_cases DESC;
```

### Query 4: Case-fatality ratio in Italy, April 2020

```sql
SELECT
  SUM(cumulative_confirmed) AS total_confirmed_cases,
  SUM(cumulative_deceased)  AS total_deaths,
  SUM(cumulative_deceased) / SUM(cumulative_confirmed) * 100 AS case_fatality_ratio
FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE country_name = 'Italy'
  AND date BETWEEN '2020-04-01' AND '2020-04-30';
```

### Query 5: Day Italy's deaths crossed 10,000

```sql
SELECT date
FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE country_name = 'Italy'
  AND cumulative_deceased > 10000
ORDER BY date ASC
LIMIT 1;
```

### Query 6: Days with zero new cases in India

```sql
WITH india_cases_by_date AS (
  SELECT date, SUM(cumulative_confirmed) AS cases
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE country_name = 'India'
    AND date BETWEEN '2020-02-21' AND '2020-03-15'
  GROUP BY date
),
india_previous_day_comparison AS (
  SELECT
    date,
    cases,
    LAG(cases) OVER (ORDER BY date)         AS previous_day,
    cases - LAG(cases) OVER (ORDER BY date) AS net_new_cases
  FROM india_cases_by_date
)
SELECT COUNT(*) AS days_with_zero_increase
FROM india_previous_day_comparison
WHERE net_new_cases = 0;
```

### Query 7: Days when US cases grew more than 10%

```sql
WITH us_cases_by_date AS (
  SELECT date, SUM(cumulative_confirmed) AS cases
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE country_name = 'United States of America'
    AND date BETWEEN '2020-03-22' AND '2020-04-20'
  GROUP BY date
),
us_previous_day_comparison AS (
  SELECT
    date,
    cases,
    LAG(cases) OVER (ORDER BY date) AS previous_day,
    (cases - LAG(cases) OVER (ORDER BY date)) * 100 / LAG(cases) OVER (ORDER BY date) AS percentage_increase
  FROM us_cases_by_date
)
SELECT
  date                AS Date,
  cases               AS Confirmed_Cases_On_Day,
  previous_day        AS Confirmed_Cases_Previous_Day,
  percentage_increase AS Percentage_Increase_In_Cases
FROM us_previous_day_comparison
WHERE percentage_increase > 10;
```

### Query 8: Top 10 countries by recovery rate

```sql
WITH cases_by_country AS (
  SELECT
    country_name               AS country,
    SUM(cumulative_confirmed)  AS cases,
    SUM(cumulative_recovered)  AS recovered_cases
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE date = '2020-05-10'
  GROUP BY country_name
),
recovered_rate AS (
  SELECT country, cases, recovered_cases, recovered_cases * 100 / cases AS recovery_rate
  FROM cases_by_country
)
SELECT country, cases AS confirmed_cases, recovered_cases, recovery_rate
FROM recovered_rate
WHERE cases > 50000
ORDER BY recovery_rate DESC
LIMIT 10;
```

### Query 9: Cumulative daily growth rate (CDGR) for France

CDGR = (last day cases ÷ first day cases)^(1 ÷ days between) − 1

```sql
WITH france_cases AS (
  SELECT date, SUM(cumulative_confirmed) AS total_cases
  FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
  WHERE country_name = 'France'
    AND date IN ('2020-01-24', '2020-05-10')
  GROUP BY date
),
summary AS (
  SELECT
    total_cases                                         AS first_day_cases,
    LEAD(total_cases) OVER (ORDER BY date)              AS last_day_cases,
    DATE_DIFF(LEAD(date) OVER (ORDER BY date), date, DAY) AS days_diff
  FROM france_cases
  LIMIT 1
)
SELECT
  first_day_cases,
  last_day_cases,
  days_diff,
  POW(last_day_cases / first_day_cases, 1 / days_diff) - 1 AS cdgr
FROM summary;
```

### Query 10: US daily cases and deaths for a Data Studio report

```sql
SELECT
  date,
  SUM(cumulative_confirmed) AS country_cases,
  SUM(cumulative_deceased)  AS country_deaths
FROM `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE country_name = 'United States of America'
  AND date BETWEEN '2020-03-15' AND '2020-04-30'
GROUP BY date
ORDER BY date;
```

Use **Explore with Data Studio** (now Looker Studio) on the result to build a time-series chart of cases and deaths.

---

## ⚠️ Notes

- Resource names, zones, regions and ports are generated per lab session. Always use the values shown in **your** lab instructions.
- These steps were written when the lab was live. Google revises labs over time, so the current version may differ.

## 📌 Disclaimer

This repository is a study reference. Attempt the lab on your own first, then compare approaches. It is not affiliated with or endorsed by Google.

## 👤 Author

**Girish Sharma** · Cloud & DevOps Engineer · [GitHub](https://github.com/codewithgarry) · [LinkedIn](https://www.linkedin.com/in/codewithgarry/)
