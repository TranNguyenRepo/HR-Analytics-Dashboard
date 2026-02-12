# HR Analytics Dashboard
An end-to-end HR analytics project that models synthetic workforce data in Snowflake using SQL and CTEs, performs exploratory data analysis (EDA), and delivers interactive hiring, turnover, and retention dashboards in Looker Studio.

# Overview
- **Extracted** and structured synthetic HR data with key workforce attributes for analysis
- **Transformed** and validated datasets in Snowflake SQL to ensure analytical consistency
- **Loaded** curated data into Looker Studio to build interactive workforce dashboards
- **Performed exploratory data analysis (EDA)** to uncover patterns in hiring, turnover, tenure, and employee sentiment

# Architecture
<img width="800" height="500" alt="ETL Pipeline and EDA Process" src="https://github.com/user-attachments/assets/cc32f3b8-a95e-4c87-bcef-cf149adb0967" />

# Pipeline Flow
- **Extract:** Generate and structure synthetic HR data with workforce attributes (demographics, hiring, tenure, exit feedback)
- **Transform:** Clean, standardize, and model data in Snowflake using SQL and CTEs for analytical consistency
- **Load**: Store curated datasets in Snowflake tables for reporting and analysis
- **Validate:** Perform data quality checks to ensure completeness and accuracy
- **EDA:** Conduct exploratory data analysis to identify patterns in hiring, turnover, tenure, and sentiment
- **Visualize:** Connect Snowflake to Looker Studio to build an interactive HR dashboard

# Prerequisites
- Snowflake account with SQL access
- Google / Looker Studio account
- SQL knowledge for data modeling

# Quick Start
## 1. Prepare Dataset
<img width="178" height="41" alt="Excel files" src="https://github.com/user-attachments/assets/a1c0c60c-6a66-4b90-bc70-ce2639552fff" />
## 2. Load & Transform:
- This section shows the SQL pipeline used to preprocess synthetic HR data, build analytical views, generate tenure calculations, create a master calendar, and perform exit survey sentiment analysis using Snowflake Cortex.

### Data Information

| Column Name           | Description                                                                 |
|-----------------------|-----------------------------------------------------------------------------|
| ORIGINAL_HIRE_DATE     | The date when the employee was first hired.                                 |
| LAST_HIRE_DATE         | The date when the employee transferred to another position/department, or same as ORIGINAL_HIRE_DATE if no transfer occurred. |
| SYN_CLIENT_NAME        | Synthetic (fake) name of the employee's client.                             |
| SYN_EMPLOYEE_ID        | Synthetic (fake) employee ID.                                               |


```sql
-- SQL code
-- Author: Tran Nguyen
-- Date: 2/8/2026
----------------------------------------
---SELECTIVE COLUMNS THAT CAN BE USED---
-------- USE CTEs TO BUILD A VIEW-------
----------------------------------------
CREATE OR REPLACE VIEW EMPLOYEE_DATA AS

WITH TEMP AS (
    SELECT 
        SYN_CLIENT_NAME, 
        SYN_COMPANY_GROUP, 
        CITY, 
        STATE, 
        COUNTRY, 
        SYN_EMPLOYEE_ID, 
        EMPLOYER_ID AS SYN_EMPLOYER_ID, 
        SYN_LOCATION_CODE, 
        POSITION_CODE_HOME,        
        POSITION_HOME, 
        EE_TYPE_CODE, 
        EE_TYPE, 
        EE_TYPE_CLASS_CODE, 
        EE_TYPE_CLASS, 
        EE_STATUS_CODE, 
        EE_STATUS, 
        SYNTHETIC_EMPLOYEE_NAME AS SYN_EMPLOYEE_NAME, 
        ORIGINAL_HIRE_DATE, 
        OLD_POSITION, 
        LAST_HIRE_DATE,
        NEW_POSITION, 
        EE_TERMINATION_DATE, 
        TERMINATION_REASON, 
        EMPLOYEE_POSITION, 
        POSITION_CODE_HOME_COM, 
        SYN_LOCATION_HOME_CODE AS SYN_FACILITY_HOME, 
        TERMINATION_TYPE
    FROM TURNOVER_DASHBOARD.PUBLIC.SYNTHETIC_EMPLOYEE_DATA
    WHERE ORIGINAL_HIRE_DATE >= '01/01/2020'
)

SELECT * 
FROM TEMP
WHERE 
    (
        EE_STATUS_CODE = 'A'
        AND EE_TERMINATION_DATE IS NULL
    )
    OR
    (
        EE_STATUS_CODE = 'L'
    )
    OR 
    (
        EE_STATUS_CODE = 'T'
        AND EE_TERMINATION_DATE IS NOT NULL
    ); -- data is not fully updated with EE_TERMINATION_DATE, we will remove those rows to avoid confusion

---------------------------------------------------------------------------------------------------------------------------------------------------
------------------CHECK IF ORIGINAL HIRE DATE != LAST HIRE DATE WHEN TERMINATION IS NULL > CREATE TRANSFERRED DATE COLUMNS-------------------------
----------------------------------------------------ADD AGGREGATION COLUMNS: TENURE----------------------------------------------------------------
---------------------------------------------------------------------------------------------------------------------------------------------------
CREATE OR REPLACE VIEW EMPLOYEE_DATA_WT_TENURE_CAL AS

WITH TEMP AS (
    SELECT 
        SYN_CLIENT_NAME,
        SYN_COMPANY_GROUP, 
        CITY, 
        STATE, 
        COUNTRY, 
        SYN_EMPLOYEE_ID, 
        SYN_EMPLOYER_ID, 
        SYN_LOCATION_CODE, 
        POSITION_CODE_HOME, 
        POSITION_HOME,
        EE_TYPE_CODE, 
        EE_TYPE, 
        EE_TYPE_CLASS_CODE, 
        EE_TYPE_CLASS, 
        EE_STATUS_CODE, 
        EE_STATUS, 
        SYN_EMPLOYEE_NAME, 
        ORIGINAL_HIRE_DATE, 
        LAST_HIRE_DATE, 
        CASE 
            WHEN ORIGINAL_HIRE_DATE <> LAST_HIRE_DATE THEN LAST_HIRE_DATE 
            ELSE NULL 
        END AS TRANSFERRED_DATE, -- CREATE A 'TRANSFERRED_DATE' TO SEPARATE WITH LAST_HIRE_DATE TO AVOID CONFUSION
        NEW_POSITION,
        CASE 
            WHEN ORIGINAL_HIRE_DATE = LAST_HIRE_DATE THEN ORIGINAL_HIRE_DATE 
            ELSE NULL 
        END AS AS_IS_ORIGINAL_HIRE_DATE, 
        OLD_POSITION,
        EE_TERMINATION_DATE, 
        TERMINATION_REASON, 
        EMPLOYEE_POSITION, 
        POSITION_CODE_HOME_COM, 
        SYN_FACILITY_HOME, 
        TERMINATION_TYPE
    FROM TURNOVER_DASHBOARD.PUBLIC.EMPLOYEE_DATA
),

TENURE_CALCULATION AS (
    SELECT *,
        CASE
            WHEN EE_TERMINATION_DATE IS NULL AND AS_IS_ORIGINAL_HIRE_DATE IS NOT NULL THEN 
                DATEDIFF(DAY, AS_IS_ORIGINAL_HIRE_DATE, CURRENT_DATE()) / 365
            WHEN EE_TERMINATION_DATE IS NULL AND AS_IS_ORIGINAL_HIRE_DATE IS NULL THEN 
                DATEDIFF(DAY, TRANSFERRED_DATE, CURRENT_DATE()) / 365
            ELSE NULL 
        END AS ACTIVE_TENURE_IN_YEARS,
        CASE
            WHEN EE_TERMINATION_DATE IS NOT NULL AND AS_IS_ORIGINAL_HIRE_DATE IS NOT NULL THEN 
                DATEDIFF(DAY, AS_IS_ORIGINAL_HIRE_DATE, EE_TERMINATION_DATE) / 365
            WHEN EE_TERMINATION_DATE IS NOT NULL AND AS_IS_ORIGINAL_HIRE_DATE IS NULL THEN 
                DATEDIFF(DAY, TRANSFERRED_DATE, EE_TERMINATION_DATE) / 365
            ELSE NULL 
        END AS TERMED_TENURE_IN_YEARS
    FROM TEMP
),

STATIC_HEAD_COUNT AS (
    SELECT 
        -- TO CALCULATE THE HEADCOUNT THAT WILL NOT DYNAMICALLY CHANGE BY DATE FOR TURNOVER RATE PURPOSE
        COUNT(DISTINCT SYN_EMPLOYEE_NAME) AS HEADCOUNT
    FROM TENURE_CALCULATION
),
...
FLAG AS (
    SELECT DISTINCT
        a.*,
        TO_DATE(t.YEAR_MONTH || '-01', 'YYYY-MM-DD') AS FULL_DATE,
        t.CALENDAR_YEAR,
        CASE
            WHEN EE_TERMINATION_DATE IS NULL 
                 AND DATE_TRUNC('MONTH', ORIGINAL_HIRE_DATE) = DATE_TRUNC('MONTH', FULL_DATE)
            THEN 1
            ELSE 0 
        END AS flag
    FROM COMBINE a
    JOIN MASTER_CALENDAR t
        ON t.SYN_EMPLOYEE_NAME = a.SYN_EMPLOYEE_NAME
)

SELECT DISTINCT *,
       ROW_NUMBER() OVER (
           PARTITION BY SYN_EMPLOYEE_NAME, FULL_DATE 
           ORDER BY SYN_EMPLOYEE_NAME, FULL_DATE
       ) AS DISTINCT_FLAG
FROM FLAG
QUALIFY DISTINCT_FLAG = 1;
---------------------------------------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------CREATE MASTER CALENDAR----------------------------------------------------------------------
---------------------------------------------------------------------------------------------------------------------------------------------------
create or replace view TURNOVER_DASHBOARD.PUBLIC.MASTER_CALENDAR(
   CALENDAR_DATE,
   CALENDAR_YEAR,
   CALENDAR_QUARTER,
   CALENDAR_MONTH,
   CALENDAR_DAY,
   YEAR_MONTH,
   CALENDAR_WEEK,
   WEEKDAY_WEEKEND,
   CALENDAR_DATE_STR
) as
WITH RECURSIVE DATE_RANGE AS (
   -- Define start and end dates for your calendar
   SELECT DATE('1980-01-01') AS CALENDAR_DATE
   UNION ALL
   SELECT DATEADD(day, 1, CALENDAR_DATE)
   FROM DATE_RANGE
   WHERE CALENDAR_DATE < CURRENT_DATE  -- or use any end date like '2030-12-31'
)
SELECT
   CALENDAR_DATE,
   YEAR(CALENDAR_DATE) AS CALENDAR_YEAR,
   QUARTER(CALENDAR_DATE) AS CALENDAR_QUARTER,
   MONTH(CALENDAR_DATE) AS CALENDAR_MONTH,
   DAY(CALENDAR_DATE) AS CALENDAR_DAY,
   TO_CHAR(CALENDAR_DATE, 'YYYY-MM') AS YEAR_MONTH,
   WEEKOFYEAR(CALENDAR_DATE) AS CALENDAR_WEEK,
   CASE 
       WHEN DAYOFWEEK(CALENDAR_DATE) IN (1,7) THEN 'Weekend'
       ELSE 'Weekday'
   END AS WEEKDAY_WEEKEND,
   TO_CHAR(CALENDAR_DATE, 'YYYY-MM-DD') AS CALENDAR_DATE_STR
FROM DATE_RANGE
ORDER BY CALENDAR_DATE;

--------------------------------------------------------------------------------------------
-------------------------SEMANTIC ANALYS FOR WHY THEY LEAVE---------------------------------
--------------------------------------------------------------------------------------------
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';

CREATE OR REPLACE VIEW TURNOVER_DASHBOARD.PUBLIC.EXIT_SURVEY_ANALYSIS
AS
WITH TEMP AS (
    SELECT DISTINCT 
        ID, 
        COMPLETION_DATE, 
        EMAIL, 
        SYN_COMPANY_NAME, 
        LOCATION, 
        SYN_FIRST_NAME, 
        SYN_LAST_NAME, 
        POSITION, 
        SYN_SUPERVISOR_NAME, 
        WHAT_IS_THE_REASON_THAT_MADE_YOU_DECIDE_TO_LEAVE,
        TRIM(f.value) AS REASON_TO_LEAVE,  -- each reason as a separate row
        PLEASE_RATE_THE_LEADERSHIP_QUALITIES_OF_UPPER_MANAGEMENT_LOCAL_AND_CORPORATE, 
        COMMENTS_TO_QUESTION_12,
        REASON,
        REASON_GROUP
    FROM EXIT_SURVEY,
    LATERAL SPLIT_TO_TABLE(WHAT_IS_THE_REASON_THAT_MADE_YOU_DECIDE_TO_LEAVE, ';') f
)
SELECT *,
        round(SNOWFLAKE.CORTEX.SENTIMENT(COMMENTS_TO_QUESTION_12),2) AS COMMENTS_SENTIMENT_SCORE

FROM TEMP
WHERE LENGTH(REASON_TO_LEAVE) >= 1 -- REMOVE NULL VALUE 
order by SYN_FIRST_NAME, SYN_LAST_NAME;


SELECT * FROM TURNOVER_DASHBOARD.PUBLIC.EXIT_SURVEY_ANALYSIS;
```
