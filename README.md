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
I stored the data, identified organization and data format, and then sorted and filtered using Google Sheets. Limited dataset includes 18 CSV files. Each file contains distinct quantitative data from 33 users. It is important to note that this constitutes a relatively small sample, and no demographic information was provided, potentially introducing sampling bias. Therefore, the sample may not accurately represent a broader population. Aditionally, the dataset is not current, encompassing only 2 months of data.

## Process Phase
I will be uploading the relevant CSV files into BigQuery and applying various data cleaning techniques using SQL. This will involve checking for missing data, ensuring correct data types and formats, identifying and removing duplicates, verifying column names, and ensuring consistency within date and time columns. All these cleaning methods could also be done using Google Sheets. I will continue searching for any potential data issues throughout the entire analysis.

There was an issue uploading hourly files and daily_sleep to BigQuery due to data type, as it only accepts UTC standard type and time format was local time. Therefore, uploaded the files and edited the schema to STRING type in order to create the table. **Fixed this issue later on by using the SUBSTR function and removing the necessary part of the string.**

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

After a thorough inspection, **33** shared unique user ids were found for daily_activity, daily_calories, hourly_calories and hourly_steps. The dataset for daily_sleep has **24** user ids. Also, heart_rate has **7** user ids, and weight_log has **8** user ids; I won't be considering those datasets for my analysis due to having a very small sample. Despite the minimal sample size requirement being 30, and daily_sleep only having **24** user ids, I still intend to use the dataset for practice purposes as I am curious to see what the data will show. As the sample size decreases, the confidence interval becomes wider and less precise.

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
## Analyze & Share Phases
Now that the data is clean and stored properly; I will organize, format and aggregate the data. I will also perform calculations in order to learn more about the data and identify relationships and trends. I will then export the results from my queries, import them into a spreadsheet to look for trends and relationships by using pivot tables and then import to Tableau public to create visualizations. 

### Aggregating data
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
### Classifying Users

I am interested in finding and classifying users based on their daily sleep time. According to [Harvard Medical School](https://www.health.harvard.edu/womens-health/women-and-sleep-one-simple-step-to-a-longer-healthier-life), average adults should get seven to nine hours of sleep every night for optimum health and function; however, 60% of women regularly fall short of that goal. They state that this could be caused do to insomnia or another underlying condition that may require medical attention. Therefore, I will classify users to see the distribution based on meeting or not meeting the recommended sleep duration of seven hours. The following SQL query will help me do that.

```SQL
-- Query to find if users meet recommended sleep time

SELECT    
  id,   
  ROUND(AVG(total_sleep_minutes / 60), 2) AS total_sleep_hours,
 CASE
  WHEN AVG(total_sleep_minutes / 60) > 7 
  THEN 'True'
  ELSE 'False' 
 END AS meets_recommended
FROM `tribal-logic-415822.fitbit_data.daily_sleep`
GROUP BY
Id
ORDER BY
1
```
After classifying users based on their daily sleep duration, the Fitbit data below shows that 46% of users sleep more than seven hours, and 54% less than 7 hours. The results support Harvard Medical School's claim and show that the majority of women fall short of the recommended sleep hours. A bigger sample would lead to a more accurate representation.

<p align="center">
  <img src="https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/4f7df36d-391a-41bc-9c36-635176f26b03"/>
</p>

I am also interested in classifying users based on their daily steps. The [Centers for Disease Control and Prevention (CDC)](https://www.cdc.gov/diabetes/prevention/pdf/postcurriculum_session8.pdf) recommends that most adults aim for 10,000 steps per day. The average American only takes 3,000 - 4,000 steps per day. Therefore, I will classify users based on meeting or not meeting 10,000 daily steps, as this amount is considered being "active."

```SQL
-- Query to find if users meet recommended steps per day
SELECT    
  Id,
  ROUND(AVG(TotalSteps), 0) AS avg_daily_steps,  
 CASE
  WHEN AVG(TotalSteps) > 10000 
  THEN 'True'
  ELSE 'False' 
 END AS meets_steps_recommended
FROM
  `tribal-logic-415822.fitbit_data.daily_activity` 
GROUP BY
Id
ORDER BY
1
```
<p align="center">
  <img src="https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/da2bac9e-bd25-40c1-b311-8fa3ed8d2ffd"/>
</p>

The results indicate that 21% of users achieve more than 10,000 daily steps, while 79% fall short of the recommended amount. However, the statistical summaries below reveal that the average total steps per day among Fitbit users is 7,638, suggesting a generally higher activity level compared to the average American. Despite this, the majority of users do not meet the recommended daily goal of 10,000 steps, falling short approximately 2,500 steps on average.

### Statistical summaries to enhance comprehension of the data
```SQL
-- Query to create statistical summary for daily_sleep

SELECT 
  ROUND(AVG(total_sleep_minutes/60), 4) AS avg_sleep_hours,
  ROUND(MIN(total_sleep_minutes/60), 4) AS min_sleep_hours,
  ROUND(MAX(total_sleep_minutes/60), 4) AS max_sleep_hours,
  ROUND(STDDEV(total_sleep_minutes/60), 4) AS stddev_sleep_hours,
  ROUND(VARIANCE(total_sleep_minutes/60), 4) AS variance_sleep_hours
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
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/991bff89-4c2d-47bb-a5a5-332f4a8b535a)

```SQL
-- Query to create statistical summary for daily_activity

SELECT
  ROUND(AVG(TotalSteps)) AS avg_total_steps,
  MIN(TotalSteps) AS min_total_steps,
  MAX(TotalSteps) AS max_total_steps,
  ROUND(STDDEV(TotalSteps), 2) AS stddev_total_steps,
  ROUND(AVG(SedentaryMinutes), 2) AS avg_sm,
  ROUND(AVG(LightlyActiveMinutes), 2) AS avg_lam,
  ROUND(AVG(FairlyActiveMinutes), 2) AS avg_fam,
  ROUND(AVG(VeryActiveMinutes), 2) AS avg_vam,
  SUM(SedentaryMinutes) AS sum_sm,
  SUM(LightlyActiveMinutes) AS sum_lam,
  SUM(FairlyActiveMinutes) AS sum_fam,
  SUM(VeryActiveMinutes) AS sum_vam,
  SUM(SedentaryMinutes + LightlyActiveMinutes + FairlyActiveMinutes + VeryActiveMinutes) AS sum_active_minutes
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
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/f6186d73-b140-4360-b2e3-7fd886f041fd)
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/0c2a9fd2-a414-4a21-89f8-914b52683c49)

```SQL
-- Query to create statistical summary for hourly_calories and hourly_steps

SELECT
  AVG(total_steps) AS avg_steps,
  MIN(total_steps) AS min_steps,
  MAX(total_steps) AS max_steps,
  STDDEV(total_steps) AS stddev_steps,
  VARIANCE(total_steps) AS var_steps,
  AVG(Calories) AS avg_calories,
  MIN(Calories) AS min_calories,
  MAX(Calories) AS max_calories,
  STDDEV(Calories) AS stddev_calories,
  VARIANCE(Calories) AS var_calories,
FROM
 `tribal-logic-415822.fitbit_data.hourly_calories` C
LEFT JOIN
 `tribal-logic-415822.fitbit_data.hourly_steps` S
ON
 C.id = S.id
 AND C.activity_hour=S.activity_hour
```
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/7aed936d-ad8f-4e96-ada4-c00afe7c9c54)
![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/f456c5bd-2623-4fd7-a69a-c16f2dc7d110)

### A few key findings from analyzing the previous data

- Avg. sleep hours per day is 6.99
- Avg. total steps per day is 7638
- Avg. daily sedentary minutes is 991, or 16.5 hours
- Avg. daily very active minutes is 21
- High standard deviations reflect a large amount of variation in the sample being studied


### Users' daily activity distribution
The pie chart below depicts the distribution of users' daily activities based on their activity type durations. It reveals that 81% of users' time is dedicated to sedendary activities, indicating a lifestyle characterized by prolonged sitting and lying down, resulting in lower calorie expenditure. Additionally, the chart indicates that 16% of their time is spent lightly active, 1% fairly active, and 2% very active. Relating this data to the statistical summary presented above, the average daily sedentary time for users is 991 minutes, or 16.5 hours, while the average very active time is only 21 minutes. Also, the majority of the users spend their time doing light activities rather than very active. 

![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/6be7e5ca-0427-4110-89c3-dccf2cbc2ed5)

### Hourly average steps
The following chart illustrates a breakdown of the hourly average steps users take throughout the day. It shows that users are most active between 5-8 PM, followed by 12-3 PM. This indicates that users are more active around lunchtime and reach their peak activity after leaving work in the evening. I believe the highest activity spike is due to most people exercising in the evening rather than the morning.

![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/4d4ef404-387f-43c8-92e3-8f2ce92ae24d)

### Total steps per weekday
The next chart displays the total steps per weekday. It reveals that Tuesday sees the highest activity levels among users, followed by a gradual decline on Wednesday and Thursday. Additionally, Saturday shows as the most active weekend day, in contrast to Sunday, which ranks as the least active day of the week.

![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/469cad4f-93ed-4e7e-a24d-3ae4e5443a42)

### Relationship between daily steps and total calories
The next chart shows the relationship between daily steps and total calories burned. Following a regression analysis, a positive linear correlation with an R-squared value of .33 can be observed. This value suggests that the data points are moderately dispersed from the regression line as the R-squared value is closer to 0. The upward slope of the relationship indicates that as the daily steps increase, there is a consistent increase in calories burned.

![image](https://github.com/valladaresr/Google-Case-Study-Bellabeat/assets/163466485/7f74ee17-e5fc-4773-964d-4f4ad2f15895)

## Act Phase
Bellabeat is a company that empowers women with knowledge about their own health and habits, and helps them make healthy decisions. Analyzing trends and relationships in the Fitbit data offers valuable insights about the users’ sleep patterns and activity levels that can be applied to Bellabeat’s customers, given their shared interest in health and fitness among their customer base. 

The following insights and recommendations could help influence Bellabeat’s marketing strategy:

| Recommendation | Description  |
| ------- | --- |
| Sleep duration | Due to the sleep deficiency observed among users, indicated by the majority (54%) falling short of the recommended 7 hours, we recommend **integrating sleep features such as alarms, to assist users in planning their sleep schedules and improve sleep quality**. |
| Step count | Despite a higher average step count among the Fitbit users compared to the national average, there’s a significant gap of approximately 2,500 steps in meeting the recommended daily step count of 10,000. We propose **incentivizing users to achieve the daily goal of 10,000 steps by offering challenges and rewards as motivation**.  |
| Activity levels | A key finding was that 81% of the users’ total time is dedicated to sedentary activities, with an average of 16.5 hours daily. Sedentary behavior predominates in users’ daily activities, indicating a need to promote more active lifestyles. We suggest **implementing reminders to encourage movement after detecting prolonged sedentary activity and break the sedentary cycle**.  |
| Hourly activity | Users exhibit highest activity levels between 5-8 PM. Given the prevalence of a 9-5 work pattern, we recommend **offering events during these peak periods to engage users effectively,** and **developing personalized activity plans tailored to users’ preferences and schedules.**
| Weekday activity  | Activity levels peak from Tuesday to Thursday, while reduced activity can be observed from Friday to Monday. Therefore, we suggest **offering targeted incentives during the less active period of the week to promote physical activity**. |
| Calories burned | Educate users about the correlation between daily step count and calories burned to empower them to make informed decisions about their activity levels and dietary choices. Our recommendation is to **foster a supportive and motivating community environment where users can share their achievements and support to each other**. |

By implementing these insights and recommendations, Bellabeat can enhance its platform’s effectiveness in promoting active lifestyles and improving users’ overall health. I recommend Bellabeat to continuously gather and analyze its own company data and feedback from customers, to monitor changes over time and assess the impact of the implemented strategies on user behavior.
