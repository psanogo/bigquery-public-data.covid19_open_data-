Task 1: Total confirmed cases
This query sums the cumulative_confirmed cases for the specified date across all locations to give a global total.

#standardSQL
SELECT
  SUM(cumulative_confirmed) AS total_cases_worldwide
FROM
  `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE
  date = '2020-05-15'


  
Task 2: Worst affected areas
To find the number of states in the US with a significant number of deaths, we filter by country and date, then count the distinct states (subregion1_name) that meet the death toll criteria. We also ensure we don't count any entries where the state name is NULL.



sql
 Show full code block 
#standardSQL
SELECT
  COUNT(DISTINCT subregion1_name) AS count_of_states
FROM
  `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE
  country_name = 'United States of America'
  AND date = '2020-05-15'
  AND cumulative_deceased > 100
  AND subregion1_name IS NOT NULL

  
Task 3: Identify hotspots
This query selects the name and confirmed cases for US states exceeding a certain threshold on a specific date. The results are ordered to show the most affected states first.



sql
 Show full code block 
#standardSQL
SELECT
  subregion1_name AS state,
  cumulative_confirmed AS total_confirmed_cases
FROM
  `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE
  country_code = 'US'
  AND date = '2020-05-15'
  AND cumulative_confirmed > 1000
  AND subregion1_name IS NOT NULL
ORDER BY
  total_confirmed_cases DESC

  
Task 4: Fatality ratio
To calculate the case-fatality ratio for Italy at the end of May 2020, we find the cumulative totals on the last day of the month and then perform the calculation. Using SUM ensures we get a single, country-wide total even if the data has sub-region entries.



sql
 Show full code block 
#standardSQL
SELECT
  SUM(cumulative_confirmed) AS total_confirmed_cases,
  SUM(cumulative_deceased) AS total_deaths,
  (SUM(cumulative_deceased) * 100) / SUM(cumulative_confirmed) AS case_fatality_ratio
FROM
  `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE
  country_name = 'Italy'
  AND date = '2020-05-31'

  
Task 5: Identifying specific day
To find the first day the death toll crossed a specific number, we can aggregate the deaths by date for Italy, filter for days exceeding the threshold, and then select the earliest date from that set.



sql
 Show full code block 
#standardSQL
WITH
  daily_deaths_italy AS (
    SELECT
      date,
      SUM(cumulative_deceased) AS total_deceased
    FROM
      `bigquery-public-data.covid19_open_data.covid19_open_data`
    WHERE
      country_name = 'Italy'
    GROUP BY
      date
  )
SELECT
  date
FROM
  daily_deaths_italy
WHERE
  total_deceased > 8000
ORDER BY
  date
LIMIT 1


Task 6: Finding days with zero net new cases
The provided query was a good start! It correctly calculated the daily new cases but was missing the final step. The corrected query below adds a final SELECT statement to count the number of days where the net_new_cases value is zero.



sql
 Show full code block 
#standardSQL
WITH
  india_cases_by_date AS (
    SELECT
      date,
      SUM(cumulative_confirmed) AS cases
    FROM
      `bigquery-public-data.covid19_open_data.covid19_open_data`
    WHERE
      country_name = "India"
      AND date BETWEEN '2020-02-21' AND '2020-03-11'
    GROUP BY
      date
  ),
  india_previous_day_comparison AS (
    SELECT
      date,
      cases,
      LAG(cases, 1) OVER (ORDER BY date) AS previous_day_cases,
      cases - LAG(cases, 1) OVER (ORDER BY date) AS net_new_cases
    FROM
      india_cases_by_date
  )
SELECT
  COUNTIF(net_new_cases = 0) AS days_with_zero_increase
FROM
  india_previous_day_comparison

  
Task 7: Doubling rate
Using a similar structure with Common Table Expressions (CTEs), this query first aggregates daily cases for the US, then uses the LAG function to compare each day to the previous one, calculating the percentage increase. The final SELECT filters for dates where this increase was greater than 10%.



sql
 Show full code block 
#standardSQL
WITH
  us_cases_by_date AS (
    SELECT
      date,
      SUM(cumulative_confirmed) AS cases
    FROM
      `bigquery-public-data.covid19_open_data.covid19_open_data`
    WHERE
      country_name = "United States of America"
      AND date BETWEEN '2020-03-22' AND '2020-04-20'
    GROUP BY
      date
  ),
  us_daily_increase AS (
    SELECT
      date,
      cases AS Confirmed_Cases_On_Day,
      LAG(cases, 1) OVER (ORDER BY date) AS Confirmed_Cases_Previous_Day
    FROM
      us_cases_by_date
  )
SELECT
  date AS Date,
  Confirmed_Cases_On_Day,
  Confirmed_Cases_Previous_Day,
  (
    (
      Confirmed_Cases_On_Day - Confirmed_Cases_Previous_Day
    ) * 100 / Confirmed_Cases_Previous_Day
  ) AS Percentage_Increase_In_Cases
FROM
  us_daily_increase
WHERE
  Confirmed_Cases_Previous_Day IS NOT NULL
  AND (
    (
      Confirmed_Cases_On_Day - Confirmed_Cases_Previous_Day
    ) * 100 / Confirmed_Cases_Previous_Day
  ) > 10
ORDER BY
  date

  
Task 8: Recovery rate
This query calculates the recovery rate for countries with a high number of cases. It filters for the latest data up to May 10, 2020, aggregates the numbers per country, and then filters for countries with over 50,000 confirmed cases before calculating the rate and ordering the results.



sql
 Show full code block 
#standardSQL
WITH
  country_totals AS (
    SELECT
      country_name,
      SUM(cumulative_confirmed) AS confirmed_cases,
      SUM(cumulative_recovered) AS recovered_cases
    FROM
      `bigquery-public-data.covid19_open_data.covid19_open_data`
    WHERE
      date = '2020-05-10'
    GROUP BY
      country_name
  )
SELECT
  country_name AS country,
  recovered_cases,
  confirmed_cases,
  (recovered_cases * 100) / confirmed_cases AS recovery_rate
FROM
  country_totals
WHERE
  confirmed_cases > 50000
ORDER BY
  recovery_rate DESC
LIMIT 10


Task 9: CDGR - Cumulative daily growth rate
The original query had a syntax error in the final SELECT statement. The SQRT function in SQL takes only one argument, whereas the formula for Nth root requires using the POWER or ^ operator. The corrected query uses POWER(base, exponent) to correctly calculate the CDGR.


sql
 Show full code block 
#standardSQL
WITH
  france_cases AS (
    SELECT
      date,
      SUM(cumulative_confirmed) AS total_cases
    FROM
      `bigquery-public-data.covid19_open_data.covid19_open_data`
    WHERE
      country_name = "France"
      AND date IN ('2020-01-24', '2020-05-15')
    GROUP BY
      date
  ),
  summary AS (
    SELECT
      total_cases AS first_day_cases,
      LEAD(total_cases, 1) OVER (ORDER BY date) AS last_day_cases,
      date AS first_day_date,
      LEAD(date, 1) OVER (ORDER BY date) AS last_day_date
    FROM
      france_cases
    LIMIT 1
  )
SELECT
  first_day_cases,
  last_day_cases,
  DATE_DIFF(last_day_date, first_day_date, DAY) AS days_diff,
  POWER(
    (last_day_cases / first_day_cases),
    (1 / DATE_DIFF(last_day_date, first_day_date, DAY))
  ) - 1 AS cdgr
FROM
  summary

  
Task 10. Create a Looker Studio report
This task requires actions outside of a single SQL query. Here is the step-by-step process:

Write the Base Query in BigQuery: First, you need a query that pulls the required data: daily confirmed cases and deaths for the United States within the specified date range.



sql
 Show full code block 
#standardSQL
SELECT
  date,
  SUM(cumulative_confirmed) AS Number_of_Confirmed_Cases,
  SUM(cumulative_deceased) AS Number_of_Deaths
FROM
  `bigquery-public-data.covid19_open_data.covid19_open_data`
WHERE
  country_name = 'United States of America'
  AND date BETWEEN '2020-03-25' AND '2020-04-20'
GROUP BY
  date
ORDER BY
  date;
Connect BigQuery to Looker Studio:

Go to the Looker Studio homepage.
Click Create > Data Source.
Select the BigQuery connector.
Navigate to your project, then select Custom Query.
Select the billing project where the query will run.
Paste the SQL query from Step 1 into the editor.
Click Connect in the upper right.
Build the Report:

On the data source schema page, click Create Report.
Looker Studio will create a blank report with your data source.
Go to Insert > Time series chart.
Drag the chart to the desired size on the canvas.
In the chart's Setup panel on the right:
Set Dimension to date.
Add two Metrics: Number_of_Confirmed_Cases and Number_of_Deaths.
Your chart will now plot both metrics over the specified date range, matching the requirements.
This process allows you to create a dynamic report based on the results of your BigQuery query.


