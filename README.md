# Road-accidents-compared-to-speed
--------------------------------
### WORK IN PROGRESS

## The problem: 

At the beginning of 2020, speed limits were reduced on 38 road sections in an effort to address the serious issue of car accidents.

Now, five years later, these changes are being reconsidered and many speed limits are being reverted. This analysis aims to determine whether the original reduction in speed limits had any impact on the number of crashes or the severity of injuries resulting from them.

The analysis draws on two large datasets. The primary source is the Crash Analysis System (CAS), which records crashes on all New Zealand roads and any areas where the public has legal access by motor vehicle. The second dataset is the National Speed Limit Register, which provides detailed layers of certified speed limit records across the country.

https://catalogue.data.govt.nz/dataset/crash-analysis-system-cas-data5
https://catalogue.data.govt.nz/dataset/national-speed-limit-register-nslr1 


## Exploratory Data Analysis 

### Cleaning the data involved the following steps:
Removing data points with zero or null values
Creating counts for key columns such as speed limits and injury types to identify and remove outliers
Sorting by date to identify any entries incorrectly listed as coming into effect in 2050


### Next, queries were created to analyse the data and identify any patterns or results that showed unusually strong effects


#### Crashes and the amount of Fatal crashes the happened for each speed limit
```sql
WITH crash_buckets AS (
    SELECT
        CASE
            WHEN speedlimit BETWEEN 10 AND 20 THEN '10–20 km/h'
            WHEN speedlimit BETWEEN 30 AND 40 THEN '30–40 km/h'
            WHEN speedlimit = 50 THEN '50 km/h'
            WHEN speedlimit BETWEEN 60 AND 70 THEN '60–70 km/h'
            WHEN speedlimit BETWEEN 80 AND 90 THEN '80–90 km/h'
            WHEN speedlimit BETWEEN 100 AND 110 THEN '100–110 km/h'
            ELSE 'Other / Unknown'
        END AS speed_bucket
    FROM
        road_events2
    WHERE
        fatalcount > 0
        AND speedlimit IS NOT NULL)

SELECT
    speed_bucket,
    COUNT(*) AS fatal_crash_count
FROM
    crash_buckets
GROUP BY
    speed_bucket
ORDER BY
    CASE
        WHEN speed_bucket = '10–20 km/h' THEN 1
        WHEN speed_bucket = '30–40 km/h' THEN 2
        WHEN speed_bucket = '50 km/h' THEN 3
        WHEN speed_bucket = '60–70 km/h' THEN 4
        WHEN speed_bucket = '80–90 km/h' THEN 5
        WHEN speed_bucket = '100–110 km/h' THEN 6
        ELSE 7
    END;
```

#### Comparing minor crashes and major crashes that happened in each speed limit.
```sql
WITH crash_buckets AS (
    SELECT
        CASE
            WHEN speedlimit BETWEEN 10 AND 20 THEN '10–20 km/h'
            WHEN speedlimit BETWEEN 30 AND 40 THEN '30–40 km/h'
            WHEN speedlimit = 50 THEN '50 km/h'
            WHEN speedlimit BETWEEN 60 AND 70 THEN '60–70 km/h'
            WHEN speedlimit BETWEEN 80 AND 90 THEN '80–90 km/h'
            WHEN speedlimit BETWEEN 100 AND 110 THEN '100–110 km/h'
        END AS speed_bucket,
        COALESCE(minorinjurycount, 0) AS minorinjurycount,
        COALESCE(seriousinjurycount, 0) AS seriousinjurycount,
        COALESCE(fatalcount, 0) AS fatalcount
    FROM
        road_events2
    WHERE
        speedlimit BETWEEN 10 AND 110
)

SELECT
    speed_bucket,
    SUM(minorinjurycount) AS minor_injuries,
    SUM(seriousinjurycount) AS serious_injuries,
    SUM(fatalcount) AS fatalities,
    SUM(minorinjurycount + seriousinjurycount + fatalcount) AS total_injuries
FROM
    crash_buckets
WHERE
    speed_bucket IS NOT NULL
GROUP BY
    speed_bucket
ORDER BY
    CASE
        WHEN speed_bucket = '10–20 km/h' THEN 1
        WHEN speed_bucket = '30–40 km/h' THEN 2
        WHEN speed_bucket = '50 km/h' THEN 3
        WHEN speed_bucket = '60–70 km/h' THEN 4
        WHEN speed_bucket = '80–90 km/h' THEN 5
        WHEN speed_bucket = '100–110 km/h' THEN 6
    END;
```

#### The crashes that happened at each speed limit and the amount of people and vehicles involved in those crashes.

```sql
WITH crash_buckets AS (
    SELECT
        CASE
            WHEN speedlimit BETWEEN 10 AND 20 THEN '10–20 km/h'
            WHEN speedlimit BETWEEN 30 AND 40 THEN '30–40 km/h'
            WHEN speedlimit = 50 THEN '50 km/h'
            WHEN speedlimit BETWEEN 60 AND 70 THEN '60–70 km/h'
            WHEN speedlimit BETWEEN 80 AND 90 THEN '80–90 km/h'
            WHEN speedlimit BETWEEN 100 AND 110 THEN '100–110 km/h'
        END AS speed_bucket,
        COALESCE(minorinjurycount, 0) AS minorinjurycount,
        COALESCE(seriousinjurycount, 0) AS seriousinjurycount,
        COALESCE(fatalcount, 0) AS fatalcount,

        -- Total vehicles involved in each crash
        COALESCE(bicycle, 0) +
        COALESCE(bus, 0) +
        COALESCE(moped, 0) +
        COALESCE(motorcycle, 0) +
        COALESCE(pedestrian, 0) +
        COALESCE(schoolbus, 0) +
        COALESCE(suv, 0) +
        COALESCE(taxi, 0) +
        COALESCE(truck, 0) +
        COALESCE(vehicle, 0) AS vehicles_in_crash
    FROM
        road_events2
    WHERE
        speedlimit BETWEEN 10 AND 110
)

SELECT
    speed_bucket,
    SUM(minorinjurycount) AS minor_injuries,
    SUM(seriousinjurycount) AS serious_injuries,
    SUM(fatalcount) AS fatalities,
    SUM(minorinjurycount + seriousinjurycount + fatalcount) AS total_injuries,
    SUM(vehicles_in_crash) AS total_vehicles_involved
FROM
    crash_buckets
WHERE
    speed_bucket IS NOT NULL
GROUP BY
    speed_bucket
ORDER BY
    CASE
        WHEN speed_bucket = '10–20 km/h' THEN 1
        WHEN speed_bucket = '30–40 km/h' THEN 2
        WHEN speed_bucket = '50 km/h' THEN 3
        WHEN speed_bucket = '60–70 km/h' THEN 4
        WHEN speed_bucket = '80–90 km/h' THEN 5
        WHEN speed_bucket = '100–110 km/h' THEN 6
    END;
```
#### The average speed limit of the country in each year.

```sql
----- all data put into years for the average speed of the whole country
WITH active_limits AS (
    SELECT
        speed_limit_value,
        when_effective,
        COALESCE(when_ineffective, CURRENT_DATE) AS effective_until
    FROM national_speed_limits1
    WHERE when_effective <= CURRENT_DATE
      AND speed_limit_reason IN ('Default Area', 'Speed Area', 'Urban Traffic Area')
),
yearly_limits AS (
    SELECT
        EXTRACT(YEAR FROM generate_series(when_effective, effective_until, '1 year'::interval)) AS year,
        speed_limit_value
    FROM active_limits
)
SELECT
    year,
    SUM(speed_limit_value) AS total_speed_limit,
    AVG(speed_limit_value) AS average_speed_limit,
    COUNT(*) AS record_count
FROM yearly_limits
WHERE year BETWEEN 2005 AND 2025
GROUP BY year
ORDER BY year;
```

#### The amount of crashes in each year.

```sql

SELECT
    crashyear,

    -- Sum of all vehicle types involved in crashes
    SUM(COALESCE(bicycle, 0) +
        COALESCE(bus, 0) +
        COALESCE(moped, 0) +
        COALESCE(motorcycle, 0) +
        COALESCE(pedestrian, 0) +
        COALESCE(schoolbus, 0) +
        COALESCE(suv, 0) +
        COALESCE(taxi, 0) +
        COALESCE(truck, 0) +
        COALESCE(vehicle, 0)
    ) AS total_vehicles_involved,

    -- Injury severity breakdown
    SUM(COALESCE(minorinjurycount, 0)) AS total_minor_injuries,
    SUM(COALESCE(fatalcount, 0)) AS total_fatalities,
    SUM(COALESCE(seriousinjurycount, 0)) AS total_serious_injuries,

    -- Total injuries across all severity levels
    SUM(
        COALESCE(minorinjurycount, 0) +
        COALESCE(fatalcount, 0) +
        COALESCE(seriousinjurycount, 0)
    ) AS total_injuries

FROM
    road_events2
WHERE
    crashyear IS NOT NULL
GROUP BY
    crashyear
ORDER BY
    crashyear;
```


## Case studies. 

