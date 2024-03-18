# Google Case Study - Bellabeat 

## About the Company
Bellabeat, a high-tech company that manufactures health-focused smart products. The Bellabeat app collects and provides users with health data related to their activity, sleep, stress, menstrual cycle, and mindfulness habits. This data can help users better understand their current habits and make healthy decisions. Bellabeat empowers women with knowledge about their own health and habits. 
<p align="center">
  <img src="https://github.com/valladaresr/Capstone-Bellabeat-Case-Study/assets/163466485/258bb35e-77f5-4597-bd64-a69d017688dc"/>
</p>

## Ask Phase
### Business Task
Analyze smart device usage data to identify trends in how consumers are using non-Bellabeat products and provide insights into Bellabeat’s marketing strategy.

Questions to answer for stakeholders:
1. What are some trends in smart device usage?
2. How could these trends apply to Bellabeat customers?
3. How could these trends help influence Bellabeat marketing strategy?

## Prepare Phase
### Dataset Source & Information
[FitBit Fitness Tracker Data](https://www.kaggle.com/datasets/arashnic/fitbit) (CC0: Public Domain, dataset made available through Mobius): This Kaggle data set contains personal fitness tracker from thirty fitbit users. Thirty eligible Fitbit users consented to the submission of personal tracker data, including minute-level output for physical activity, heart rate, and sleep monitoring. It includes information about daily activity, steps, and heart rate that can be used to explore users’ habits.

### Dataset Credibility and Observations
Data was stored, identified organization, data format, and then sorted and filtered using Google Sheets. Limited dataset includes 18 CSV files. Each file contains different quantitative data from 33 users. It is a small sample and there was no demographic information provided, so we could be facing a sampling bias. Therefore, the sample might not be representative of the population as a whole. Also, dataset is not current and there are only 2 months of data available.
# Add about 30 is the minimal sample, and discuss confidence interval.

## Process Phase
I will be uploading the CSV files that are useful in BigQuery and using cleaning techniques in SQL. I will be checking for missing data, data type and format, duplicates, column names, and consistency with date and time columns. It could also be easily cleaned by fixing issues like formatting dates and removing duplicates, and filtering data with a pivot table through Google Sheets. I will continue searching for issues with the data throughout the entire analysis.

There was an issue uploading hourly files and daily_sleep to BigQuery due to data type, as it only accepts UTC standard type and time format was local time. Therefore, uploaded the files and edited the schema to STRING type to be able to create the table. **Fixed this issue later by using the SUBSTR function and removing the necessary part of the string.**

I will be importing the CSV files to BigQuery that contain the following tables:
- daily_activity
- daily_calories
- daily_sleep
- hourly_calories
- hourly_steps
- weight_log
- heart_rate

### Checking which column names are shared across tables
```SQL
SELECT
 column_name,
 COUNT(table_name)
FROM
 `tribal-logic-415822.fitbit_data.INFORMATION_SCHEMA.COLUMNS`
GROUP BY
 1;
```
All tables share column name Id.
```SQL
SELECT
 table_name,
 SUM(CASE
     WHEN column_name = "id" THEN 1
   ELSE
   0
 END
   ) AS id_column
FROM
 `tribal-logic-415822.fitbit_data.INFORMATION_SCHEMA.COLUMNS`
GROUP BY
 1
ORDER BY
 1 ASC;
```
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/6f818fa9-feaa-4793-9bfe-17e78bb19413)

### Checking for number of distinct user ids on datasets
Counting number of distinct ids for all the 7 datasets.
```SQL
-- Example of counting number of distinct ids for daily_activity
SELECT
  COUNT(DISTINCT id) AS ids_total
FROM
  `tribal-logic-415822.fitbit_data.daily_activity`
```
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/20211613-87ca-4f44-b4a9-223d80163609)

```SQL
-- Counting number of distinct ids in daily_sleep
SELECT
  COUNT(Distinct id) AS ids_total
FROM
    `tribal-logic-415822.fitbit_data.daily_sleep`
```
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/a23c0f1b-c7fa-49fe-bec1-b1c036d50f87)

```SQL
-- Counting number of distinct ids in weight_log
SELECT
  COUNT(DISTINCT id) AS ids_total
FROM
  `tribal-logic-415822.fitbit_data.weight_log`
```
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/bdf15630-a8a8-44da-bb5e-0cf7a6169f65)

```SQL
-- Counting number of distinct ids in heart_rate
SELECT
  COUNT(DISTINCT id) AS ids_total
FROM
  `tribal-logic-415822.fitbit_data.heart_rate`
```
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/89810b0b-8e14-4faf-8187-e86dfb8007cb)

After thorough inspection, **33** user ids were found for daily_activity, daily_calories, hourly_calories and hourly_steps. The dataset for daily_sleep has **24** user ids. Also, heart_rate has **7** user ids, and weight_log has **8** user ids; I won't be considering those datasets for my analysis due to having a very small sample. 

### Checking for duplicates
```SQL
-- Checking for duplicates in daily_sleep
SELECT  
  id, 
  date AS date_time,
  total_sleep_minutes,
  total_bed_minutes,  
FROM `tribal-logic-415822.fitbit_data.daily_sleep`
GROUP BY
  id,
  date_time,
  total_bed_minutes,
  total_sleep_minutes
HAVING Count(*) > 1;
```
I found **3** duplicates in daily_sleep and those will be removed from the query when I do the analysis. The rest of the datasets look good.

![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/2cd25d40-20bc-4662-9366-ffb2d5e98be8)

### Removing duplicates
Removed duplicated rows from daily_sleep, used WITH to create temporary table then used SUBSTR function to remove static timestamp 12:00:00 AM/PM from all dates; to be able to compare with daily_activity and daily_calories datasets
```SQL
WITH fixed_date_sleep AS (  
SELECT      
DISTINCT 
  *,      
SUBSTR(date, 1, 9) AS sleep_date,    
FROM 
  `tribal-logic-415822.fitbit_data.daily_sleep`)
SELECT    
  id,   
  sleep_date,   
  total_sleep_minutes,   
  total_bed_minutes
FROM 
  fixed_date_sleep 
ORDER BY   
  id,
  fixed_date_sleep.sleep_date
```
## Analysis Phase
Now that the data is clean and stored properly; I will organize, format and aggregate the data. I will also perform calculations in order to learn more about the data and identify relationships and trends. I will then export the results from my queries, import them into a spreadsheet to look for trends and then import to Tableau public to create visualizations. 

```SQL
-- First part of query to double check that both tables share same data types, second part of query using JOIN statement to combine relevant data between daily_activity, daily_calories, and daily_sleep
SELECT
 column_name,
 table_name,
 data_type
FROM
  `tribal-logic-415822.fitbit_data.INFORMATION_SCHEMA.COLUMNS`
WHERE
 REGEXP_CONTAINS(LOWER(table_name),"daily")
 AND column_name IN (
 SELECT
   column_name
 FROM
  `tribal-logic-415822.fitbit_data.INFORMATION_SCHEMA.COLUMNS`
 WHERE
   REGEXP_CONTAINS(LOWER(table_name),"daily")
 GROUP BY
   1
 HAVING
   COUNT(table_name) >=1)
ORDER BY
  1;
SELECT
  A.Id,
  A.Calories,
  A.TotalSteps,
  A.TotalDistance,
  A.ActivityDate,
  A.SedentaryActiveDistance,
  A.LightActiveDistance,
  A.ModeratelyActiveDistance,
  A.VeryActiveDistance,
  A.SedentaryMinutes,
  A.LightlyActiveMinutes,
  A.FairlyActiveMinutes,
  A.VeryActiveMinutes,
  S.total_sleep_minutes,
  S.total_bed_minutes,
FROM
`tribal-logic-415822.fitbit_data.daily_activity` A
LEFT JOIN
`tribal-logic-415822.fitbit_data.daily_calories` C
ON
 A.Id = C.Id
 AND A.ActivityDate=C.ActivityDay
 AND A.Calories = C.Calories
LEFT JOIN
`tribal-logic-415822.fitbit_data.daily_sleep` S
ON
 A.Id = S.Id
 AND A.ActivityDate=S.sleep_date;
```



```SQL
-- First part of query to double check that both tables share same data types, second part of query using JOIN statement to combine relevant data between hourly_calories and hourly_steps
SELECT
 column_name,
 table_name,
 data_type
FROM
  `tribal-logic-415822.fitbit_data.INFORMATION_SCHEMA.COLUMNS`
WHERE
 REGEXP_CONTAINS(LOWER(table_name),"hourly")
 AND column_name IN (
 SELECT
  column_name
 FROM
  `tribal-logic-415822.fitbit_data.INFORMATION_SCHEMA.COLUMNS`
 WHERE
  REGEXP_CONTAINS(LOWER(table_name),"hourly")
 GROUP BY
  1
 HAVING
  COUNT(table_name) >=1)
ORDER BY
  1;
SELECT
  C.id,
  C.activity_hour,
  C.Calories,
  S.total_steps,
FROM
  `tribal-logic-415822.fitbit_data.hourly_calories` C
LEFT JOIN
  `tribal-logic-415822.fitbit_data.hourly_steps` S
ON
  C.id = S.id
  AND C.activity_hour=S.activity_hour
ORDER BY
  1;
```

```SQL

```

```SQL

```






- Preview tables and checking the data type
- verifying number of users 
- checking for duplicates
- removing duplicates
- checking for missing data
- consistency date and time columns
  
- merge daily_activity and daily_sleep to check for correlation, maybe more activity= more sleep.



