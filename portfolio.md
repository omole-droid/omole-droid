# Data Analysis Portfolio

## Introduction
My name is Omole Oluwaseun. I’m a data analyst who turns raw data into useful information. I have skills in statistical analysis and data visualization. I solve complex problems and help teams make decisions with data. This portfolio includes my work from the Google Data Analysis Certification Capstone project. I used SQL and Google Sheets to complete it.

## Capstone Project: Google Data Analysis Certification

### Case Study
- **Title**: Cyclistic Bike-Share Analysis
- **Description**: Cyclistic is a fictional bike-share company based in Chicago. The datasets for this project were made available by Motivate International Inc. under this license.

### Problem Statement
- **Problem Statement**: I am a junior data analyst working in the marketing analyst team at Cyclistic, a fictitious bike-share company in Chicago. The director of marketing believes the company’s future success depends on maximizing the number of annual memberships. Therefore, my team wants to understand how casual riders and annual members use Cyclistic bikes differently. From these insights, my team will design a new marketing strategy to convert casual riders into annual members. But first, Cyclistic executives must approve my recommendations, so they must be backed up with compelling data insights and professional data visualizations.

### Business Task and Objectives
- **Business Task**: The business task is to understand how annual members and casual riders use Cyclistic bikes differently. By analyzing the historical bike trip data, I aim to identify patterns and trends that distinguish the behavior of annual members from that of casual riders. The insights gained from this analysis will be crucial in designing a new marketing strategy aimed at converting casual riders into annual members, ultimately maximizing the number of annual memberships and contributing to Cyclistic’s future growth. The analysis will be presented in a report, which will include documentation of data sources, data cleaning/manipulation steps, visualizations, key findings, and top three recommendations based on the analysis.

### Data Sources and Collection
I will be using Cyclistic’s historical trip data from January 2022 to December 2022 to analyze and identify trends. The data has been made available by Motivate International Inc. under this license. This public data can be used to explore how different customer types are using Cyclistic bikes. However, data-privacy issues prohibit the use of riders’ personally identifiable information. Hence I will not be able to determine if casual riders live in the Cyclistic service area or if they have purchased multiple single passes. The data has been processed to remove trips that are taken by Cyclistic’s staff as they service and inspect the system; and any trips that were below 60 seconds in length (potentially false starts or users trying to re-dock a bike to ensure it was secure).

### Data Cleaning and Preparation
The following steps were taken to clean and process the data:
- Removed duplicate rows using the ride_id field.
- Created the following new columns:
  - Date
  - Month
  - Year
  - Day of the week
  - Hour of the day
  - Route

Filtered out rides with lengths less than a minute (i.e., below 60 seconds) and those above 24 hours (i.e., above 1440 minutes) using the script below:

```sql
-- Create new table
CREATE TABLE IF NOT EXISTS ride_data.year_ride_2022_final AS
-- Create new columns for date, month, year, day of the week, hour of the day, route
SELECT 
  DISTINCT ride_id, rideable_type, started_at, ended_at, start_station_id, start_station_name, end_station_id, end_station_name, start_lat, start_lng, end_lat, end_lng,
  IF(end_station_name = start_station_name, 1, 0) AS round_trip,
  CAST(started_at AS DATE) AS date,
  FORMAT_DATE('%b', started_at) AS month,
  FORMAT_DATE('%Y', started_at) AS year,
  TIMESTAMP_DIFF(ended_at, started_at, MINUTE) AS ride_length_mins,
  CONCAT(start_station_name, " to ", end_station_name) AS route,
  FORMAT_DATE('%a', started_at) AS day_of_week,
  FORMAT_DATE('%H', started_at) AS hour_of_day,
  CAST(started_at AS TIME) AS start_time,
  CAST(ended_at AS TIME) AS end_time,
  member_casual
FROM `artful-hexagon-397217.ride_data.year_ride_2022`
-- Filter off records with null start station name and null end station name and removed ride lengths below a minute and above 24 hours
WHERE
  start_station_name IS NOT NULL AND
  end_station_name IS NOT NULL AND
  start_station_id IS NOT NULL AND
  end_station_id IS NOT NULL AND 
  TIMESTAMP_DIFF(ended_at, started_at, MINUTE) > 1 AND 
  TIMESTAMP_DIFF(ended_at, started_at, HOUR) < 24
ORDER BY date, start_time;
```

### Analysis and Insights
Initial analysis of how annual members and casual riders use Cyclistic bikes differently.

#### Preliminary Summary Description of Rides by Ride Category
Initial descriptive statistics show that there were more rides from members but their average ride lengths were shorter (by almost half) than those of the casual riders.

### Data Visualization
Starting with the ride-lengths, I needed to have a visual of what kind of distribution that it followed. Hence I ran the following query to get the quartiles and identify the outliers in the distribution.

```sql
-- To calculate the quartiles of the ride_length distribution
-- Values above Q3 + 1.5xIQR or below Q1 - 1.5xIQR are considered as outliers. 
-- Values above Q3 + 3xIQR or below Q1 - 3xIQR are considered as extreme points (or extreme outliers)
WITH quartiles AS (
  SELECT
    PERCENTILE_DISC(ride_length_mins, 0) OVER() AS min,
    PERCENTILE_DISC(ride_length_mins, 0.25) OVER() AS q1,
    PERCENTILE_DISC(ride_length_mins, 0.5) OVER() AS median,
    PERCENTILE_DISC(ride_length_mins, 0.75) OVER() AS q3,
    PERCENTILE_DISC(ride_length_mins, 1) OVER() AS max
  FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
  WHERE member_casual = 'casual'
  LIMIT 1
)
SELECT
  min, q1, median, q3, max,
  q3 - q1 AS iqr,
  -- Evaluate the lower outlier, if the calculated is lower than the minimum, then there is no lower outlier
  CASE
    WHEN q1 + (1.5 * (q3 - q1)) > quartiles.max THEN NULL
    ELSE q1 + (1.5 * (q3 - q1))
  END AS upper_outlier,
  -- Evaluate the upper outlier, if the calculated is higher than the minimum, then there is no lower outlier
  CASE
    WHEN q1 - (1.5 * (q3 - q1)) < quartiles.min THEN NULL
    ELSE q1 - (1.5 * (q3 - q1))
  END AS lower_outlier,
  -- Evaluate the extremely lower outlier, if the calculated is lower than the minimum, then there is no lower outlier
  CASE
    WHEN q1 + (3 * (q3 - q1)) > quartiles.max THEN NULL
    ELSE q1 + (3 * (q3 - q1))
  END AS extreme_upper_outlier,
  -- Evaluate the extremely upper outlier, if the calculated is higher than the minimum, then there is no lower outlier
  CASE
    WHEN q1 - (3 * (q3 - q1)) < quartiles.min THEN NULL
    ELSE q1 - (3 * (q3 - q1))
  END AS extreme_lower_outlier
FROM quartiles;
```

The resulting table shows the distribution of the ride lengths. It is right skewed with 75% of the rides lasting between 2 and 26 minutes. The distributions of ride lengths for all rides and for the 2 rider categories are shown in the table below:

#### Frequency Distribution Summary of Ride Lengths
From the table, it is safe to assume that ride lengths above 62 are outliers and will be treated as such.

So I used the query below to create a frequency table of the ride-lengths with bins of 2 minute-intervals:

```sql
SELECT
  CONCAT(FLOOR(ride_length_mins / 2) * 2, '-', FLOOR(ride_length_mins / 2) * 2 + 1) AS ride_length_bin,
  COUNT(*) AS bin_count
FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
-- Change member_casual to "member" or "casual"
WHERE member_casual = 'member'
GROUP BY ride_length_bin
ORDER BY MIN(ride_length_mins);
```

The histogram obtained from the resulting table is shown in Chart 1 below:

#### Chart 1: Ride-Length Frequency Distribution for All Rides, Member Rides, and Casual Rides
![Ride-Length Frequency Distribution](images/ride_length_distribution.png)

The distribution of the ride lengths (excluding outliers) are shown above with member ride-lengths of 4–7 minutes being the most frequent and casual rides lasting between 6 and 11 minutes were the most popular in that category. The area in which these riders live may shed some light on why we have this pattern as well as their occupation.

Do these rides (length and frequency) change with day of the week? To analyze this, I used the query below to get aggregates of the ride-length and frequencies by day of the week:

```sql
-- To get ride summary statistics by weekday
SELECT  
  day_of_week, 
  COUNT(DISTINCT ride_id) AS number_of_rides,
  AVG(ride_length_mins) AS average_ridelength,
  MAX(ride_length_mins) AS max_ridelength,
  MIN(ride_length_mins) AS min_ridelength
FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
-- Change the where argument to "casual" or "member" as needed
WHERE member_casual = 'casual'
GROUP BY day_of_week
ORDER BY day_of_week;
```

Merging the tables from member and casual rides we have the column chart below:

#### Charts 2: Number of Rides by Day of the Week for Both Rider Categories & Chart 3: Average Ride-Length by Day of the Week
![Number of Rides by Day of the Week](images/number_of_rides_by_day.png)
![Average Ride-Length by Day of the Week](images/average_ride_length_by_day.png)

Charts 2 and 3 above show the daily trends. The highest number of member rides were on weekdays between Monday and Friday and the length of the rides were consistent around the 12 minute mark suggesting that the weekly rides were for daily commuting to and from work or school.

The casual riders, on the other hand, have a higher number of rides during the weekends (Saturday and Sunday) and longer ride lengths on the average. This also suggests that these rides were mostly done for pleasure. The fact that member rides during weekends were also a bit longer indicates that many of the member riders who were apparently using the rides for commuting during the week, also use the bike rides for leisure during the weekends. This implies that casual riders can become members and still use the bikes for leisure rides.

Let’s analyze the rides by the hour of the day to check for any patterns. I used aggregations statements as shown in the SQL query below:

```sql
-- To get ride summary statistics by hour of the day
SELECT 
  hour_of_day, 
  COUNT(DISTINCT ride_id) AS number_of_rides,
  AVG(ride_length_mins) AS average_ridelength,
  MAX(ride_length_mins) AS max_ridelength,
  MIN(ride_length_mins) AS min_ridelength
FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
-- Change the where argument to "casual" or "member" as needed
WHERE member_casual = 'casual'
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

#### Member Ride Data by Hour of the Day & Casual Ride Data by Hour of the Day
For members, the number of rides peaks around 7 to 8am and between 4 and 6pm indicating that the rides are most likely for commuting to and from work (or school) during these hours respectively. The casual rides are also highest around the 3pm to 6pm but this trend needs to be studied on a daily basis to expose more information.

So I modified the previous query to also select, and group by, the day_of_week field. The resulting table was used to plot the charts shown below:

#### Chart 4: Number of Rides by Hour of the Day for Weekdays
![Number of Rides by Hour of the Day for Weekdays](images/rides_by_hour_weekdays.png)

As shown in the chart above, the number of member rides per hour (blue charts on the left) during weekdays peaks between 7 and 8am and then between 4 and 6pm indicating that the riders are most likely commuting to and from work with these bikes. The casual riders show a different trend which is consistent during the weekdays showing that the peaks seen between 4 and 6pm will also have contributions from workers’ commute.

#### Chart 5: Number of Rides by Hour of the Day for Weekends
![Number of Rides by Hour of the Day for Weekends](images/rides_by_hour_weekends.png)

The weekend rides for both the members and casual riders follow the same pattern where the rides are for leisure or exercise. Members, who from the patterns observed are apparently workers and/or students, also use the bikes for recreational purposes on the weekends.

Moving on to the analysis of the rides by month and by rider category, my query was quite similar to that used for the weekday. Simply made it by month:

```sql
-- To get ride summary statistics by month
SELECT
  month, 
  COUNT(DISTINCT ride_id) AS number_of_rides,
  AVG(ride_length_mins) AS average_ridelength,
  MAX(ride_length_mins) AS max_ridelength,
  MIN(ride_length_mins) AS min_ridelength
FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
-- Change the where argument to "casual" or "member" as needed
WHERE member_casual = 'member'
GROUP BY month
ORDER BY month;
```

The resulting table was used to plot the chart below:

#### Chart 6: Monthly Analysis of Number of Rides and Average Ride-Length
![Monthly Analysis of Number of Rides](images/rides_by_month.png)

The monthly chart above shows a similar trend for both casual and member rides peaking between July and August suggesting that the harsh weather conditions around the holidays are discouraging for bike riders thereby causing the drop in bike rides in that season as well as the ride lengths. The average ride length is also lower during the holidays though the changes observed over the course of the year are more noticeable with casual riders. This suggests that members are mostly people who commute regularly to and from their place of occupation (worker or student).

Next, I considered the routes for any “secrets” which may be revealed or any confirmations as the case may be.

```sql
SELECT 
  route,
  COUNT(ride_id) AS rides,
  SUM(round_trip) AS roundtrips
FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
-- Change member_casual to "member" or "casual"
WHERE member_casual = 'casual'
GROUP BY route
ORDER BY COUNT(ride_id) DESC;
```

#### Popular Routes for Casual Riders
![Popular Routes for Casual Riders](images/popular_routes_casual.png)

Looking at the top-ten start-stations by rider category, 5 out of that for the casual rides were roundtrips (start station is same as end station) these were more likely to be recreational rides either for the exercise or to enjoy the scenery. Also we can see that routes 3 & 4 as well as routes 7 & 8 are just reverse of each other suggesting that the riders were commuting to and from a regular destination like work or school.

#### Popular Routes for Member Rides
![Popular Routes for Member Rides](images/popular_routes_member.png)

The member rides on the other hand do not have any roundtrips, suggesting the member rides were for commuting. The routes which are just reverse of themselves (1 & 2, 3 & 4, 5 & 6, and 7 & 8) also seem to support this suggestion.

The casual riders who used the bikes for commuting are ‘low-hanging’ fruits as they would be more likely to switch to members if they are targeted with promos.

Also drilling down to see the number of stations which have peak number of rides on different weekdays. I used the query below for this. Actually was the most difficult code for me to write (as a beginner).

```sql
WITH DailyRideCounts AS (
  SELECT
    start_station_name AS station,
    day_of_week AS day,
    COUNT(*) AS ride_count
  FROM `artful-hexagon-397217.ride_data.year_ride_2022_final`
  -- The member_casual can be "member" or "casual"
  WHERE member_casual = 'casual'
  GROUP BY station, day
)
SELECT
  station,
  day,
  ride_count
FROM (
  SELECT
    station,
    day,
    ride_count,
    ROW_NUMBER() OVER (PARTITION BY station ORDER BY ride_count DESC) AS rank
  FROM DailyRideCounts
) AS ranked_data
WHERE rank = 1 
ORDER BY ranked_data.ride_count DESC;
```

The resulting table gave an idea on the approach marketing may use to reach the riders.

#### Days of the Week and the Number of Stations Which Have Peak Rides on Those Days
![Peak Rides by Day of the Week](images/peak_rides_by_day.png)

Members peak on weekdays and we have some stations which experience peak of members on a Saturday. Casual rides peak on weekends which further supports the idea that casual members use the rides for leisure. Promotions and adverts can be tailored such that for each station the adverts and promotions are scheduled for the days when the casual rides are expected to be highest to improve impressions and increase chances of them converting to members.

### Recommendations
1. **Weekday Promotions**:
   - Cyclistic could focus on weekday promotions or incentives to convert casual riders into members who would use the service more during the week.
   - Offer discounted student membership.
   - Bonuses to members
