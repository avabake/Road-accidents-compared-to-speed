# Road-accidents-compared-to-speed


--oube checking table names
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public';

-- inserting table
COPY "public"."road_events2"
FROM 'D:\\Data SQL project\\Road anaylsis\\Road_Events.csv'
DELIMITER ',' 
CSV HEADER
NULL AS '';


ALTER TABLE "public"."road_events2"
  ALTER COLUMN crashLocation2 TYPE TEXT USING crashLocation2::TEXT;


ALTER TABLE "public"."road_events2"
  ALTER COLUMN x TYPE TEXT USING x::TEXT,
  ALTER COLUMN y TYPE TEXT USING y::TEXT;

  
SELECT * FROM "public"."national_speed_limits1" LIMIT 10;
