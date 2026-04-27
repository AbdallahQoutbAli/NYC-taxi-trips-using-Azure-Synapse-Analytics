# NYC Taxi Trips Analytics with Azure Synapse

This project demonstrates an end-to-end data engineering solution using **Azure Synapse Analytics** and the **NYC Taxi Trips dataset**. The solution covers data discovery, data virtualization, ingestion, transformation, and reporting using Serverless SQL, external data sources, views, stored procedures, CETAS, Parquet files, and Power BI.

![NYC Taxi Services](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/8293544d-c6a4-46f6-b77b-4137b59d8693)

## 1. Data Overview

The dataset used in this project is based on New York City taxi trip records. NYC has several types of taxi and for-hire vehicle services:

- **Yellow taxis**: Mainly pick up passengers in the inner city.
- **Green taxis**: Pick up passengers in the outer boroughs and can drop them off anywhere in the city.
- **For-Hire Vehicles (FHV)**: Operate throughout the city.
- **High-Volume For-Hire Vehicles (HVFHV)**: A further classification of for-hire vehicles.

## 2. NYC Taxi Data Files Overview

The project uses multiple file types and reference datasets:

1. Taxi Zone — CSV
2. Calendar — CSV
3. Trip Type — TSV
4. Rate Code — JSON
5. Payment Type — JSON
6. Vendor — quoted CSV
7. Trip Data — Parquet, CSV, and Delta

![NYC Taxi Data Files](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/540bd0bf-bc64-4e1a-af1f-8e67cf1fa1bb)

## 3. Solution Architecture

The solution is built around Azure Synapse Serverless SQL and follows these main stages:

1. **Data Discovery** using Serverless SQL to inspect raw files and understand data quality.
2. **Data Virtualization** using external data sources and external file formats to simplify querying files directly from storage.
3. **Data Ingestion** using external tables, stored procedures, and views over partitioned files.
4. **Data Transformation** using T-SQL, CETAS, views, and stored procedures to clean, join, aggregate, and store analytical datasets in Parquet format.
5. **Reporting** using Power BI dashboards for taxi demand, payment behaviour, and operational analysis.

![Solution Architecture](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/52a2d5ed-fb72-4ad1-92c4-a0ef020c717e)

## 4. Project Requirements

### 4.1 Data Discovery

The discovery phase focuses on understanding the structure, quality, and business value of the raw data.

Key tasks:

- Identify duplicate records.
- Check for missing values.
- Detect invalid or unexpected column values.
- Join data from multiple files.
- Summarise and aggregate data.
- Apply basic transformations.

**Path:** `SQL Scripts/discovery/`

#### Assignment: Cash vs Credit Card Trips by Borough

The objective is to identify the percentage of trips paid by cash and credit card across each borough.

```sql
WITH v_payment_type AS
(
    SELECT
        CAST(JSON_VALUE(jsonDoc, '$.payment_type') AS SMALLINT) AS payment_type,
        CAST(JSON_VALUE(jsonDoc, '$.payment_type_desc') AS VARCHAR(15)) AS payment_type_desc
    FROM OPENROWSET(
        BULK 'payment_type.json',
        DATA_SOURCE = 'nyc_taxi_data_raw',
        FORMAT = 'CSV',
        PARSER_VERSION = '1.0',
        FIELDTERMINATOR = '0x0b',
        FIELDQUOTE = '0x0b',
        ROWTERMINATOR = '0x0a'
    )
    WITH
    (
        jsonDoc NVARCHAR(MAX)
    ) AS payment_type
),
v_taxi_zone AS
(
    SELECT
        *
    FROM OPENROWSET(
        BULK 'taxi_zone.csv',
        DATA_SOURCE = 'nyc_taxi_data_raw',
        FORMAT = 'CSV',
        PARSER_VERSION = '2.0',
        FIRSTROW = 2,
        FIELDTERMINATOR = ',',
        ROWTERMINATOR = '\n'
    )
    WITH (
        location_id SMALLINT 1,
        borough VARCHAR(15) 2,
        zone VARCHAR(50) 3,
        service_zone VARCHAR(15) 4
    ) AS [result]
),
v_trip_data AS
(
    SELECT
        *
    FROM OPENROWSET(
        BULK 'trip_data_green_parquet/year=2021/month=01/**',
        FORMAT = 'PARQUET',
        DATA_SOURCE = 'nyc_taxi_data_raw'
    ) AS [result]
)
SELECT
    v_taxi_zone.borough,
    COUNT(1) AS total_trips,
    SUM(CASE WHEN v_payment_type.payment_type_desc = 'Cash' THEN 1 ELSE 0 END) AS cash_trips,
    SUM(CASE WHEN v_payment_type.payment_type_desc = 'Credit card' THEN 1 ELSE 0 END) AS card_trips,
    CAST(
        (SUM(CASE WHEN v_payment_type.payment_type_desc = 'Cash' THEN 1 ELSE 0 END) / CAST(COUNT(1) AS DECIMAL)) * 100
        AS DECIMAL(5, 2)
    ) AS cash_trips_percentage,
    CAST(
        (SUM(CASE WHEN v_payment_type.payment_type_desc = 'Credit card' THEN 1 ELSE 0 END) / CAST(COUNT(1) AS DECIMAL)) * 100
        AS DECIMAL(5, 2)
    ) AS card_trips_percentage
FROM v_trip_data
LEFT JOIN v_payment_type
    ON v_trip_data.payment_type = v_payment_type.payment_type
LEFT JOIN v_taxi_zone
    ON v_trip_data.PULocationId = v_taxi_zone.location_id
WHERE v_payment_type.payment_type_desc IN ('Cash', 'Credit card')
GROUP BY v_taxi_zone.borough
ORDER BY v_taxi_zone.borough;
```

**Output:**

![Cash vs Credit Card Trips by Borough](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/415df501-86e3-433f-82f7-bc4c8d7df843)

### 4.2 Data Virtualization

Data virtualization creates a logical data layer that allows users to query files directly from storage without building complex ETL pipelines first.

#### Create External Data Source

```sql
IF NOT EXISTS (SELECT * FROM sys.external_data_sources WHERE name = 'nyc_taxi_src')
    CREATE EXTERNAL DATA SOURCE nyc_taxi_src
    WITH
    (
        LOCATION = 'Path'
    );
```

#### Create External File Format

```sql
IF NOT EXISTS (SELECT * FROM sys.external_file_formats WHERE name = 'csv_file_format')
    CREATE EXTERNAL FILE FORMAT csv_file_format
    WITH
    (
        FORMAT_TYPE = DELIMITEDTEXT,
        FORMAT_OPTIONS
        (
            FIELD_TERMINATOR = ',',
            STRING_DELIMITER = '"',
            FIRST_ROW = 2,
            USE_TYPE_DEFAULT = FALSE,
            ENCODING = 'UTF8',
            PARSER_VERSION = '2.0'
        )
    );
```

#### Full Example Using External Source and External File Format

![External Source and File Format Example](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/8a4316d1-784e-4826-9109-5c7da58c41fc)

### 4.3 Data Ingestion

The data is partitioned across different folders, so the ingestion layer combines files from multiple paths and exposes them through views. This makes the data easier to query and keeps partition columns such as `year` and `month` available for filtering.

![Partitioned Data Ingestion](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/ced5f8eb-1df3-4f99-a9ee-28d91ba829ea)

#### Create a View for Green Taxi Trip Data

```sql
DROP VIEW IF EXISTS silver.vw_trip_data_green;
GO

CREATE VIEW silver.vw_trip_data_green
AS
SELECT
    result.filepath(1) AS year,
    result.filepath(2) AS month,
    result.*
FROM OPENROWSET(
    BULK 'silver/trip_data_green/year=*/month=*/*.parquet',
    DATA_SOURCE = 'nyc_taxi_src',
    FORMAT = 'PARQUET'
)
WITH (
    vendor_id INT,
    lpep_pickup_datetime DATETIME2(7),
    lpep_dropoff_datetime DATETIME2(7),
    store_and_fwd_flag CHAR(1),
    rate_code_id INT,
    pu_location_id INT,
    do_location_id INT,
    passenger_count INT,
    trip_distance FLOAT,
    fare_amount FLOAT,
    extra FLOAT,
    mta_tax FLOAT,
    tip_amount FLOAT,
    tolls_amount FLOAT,
    ehail_fee INT,
    improvement_surcharge FLOAT,
    total_amount FLOAT,
    payment_type INT,
    trip_type INT,
    congestion_surcharge FLOAT
) AS [result];
GO

SELECT TOP (100) *
FROM silver.vw_trip_data_green;
GO
```

### 4.4 Data Transformation

The transformation layer prepares the data for analytics and reporting.

Key objectives:

- Join the key information required for reporting.
- Join the key information required for analysis.
- Query the ingested data using SQL.
- Analyse transformed data using T-SQL.
- Store transformed datasets in columnar format, such as Parquet.

#### Business Requirement 1: Credit Card Payment Campaign

The business wants to encourage more customers to use credit card payments. The analysis focuses on:

1. Trips paid by credit card vs cash.
2. Payment behaviour by weekday and weekend.
3. Payment behaviour by borough.

```sql
SELECT
    td.year,
    td.month,
    tz.borough,
    CONVERT(DATE, td.lpep_pickup_datetime) AS trip_date,
    cal.day_name AS trip_day,
    CASE WHEN cal.day_name IN ('Saturday', 'Sunday') THEN 'Y' ELSE 'N' END AS trip_day_weekend_ind,
    SUM(CASE WHEN pt.description = 'Credit card' THEN 1 ELSE 0 END) AS card_trip_count,
    SUM(CASE WHEN pt.description = 'Cash' THEN 1 ELSE 0 END) AS cash_trip_count
FROM silver.vw_trip_data_green td
JOIN silver.taxi_zone tz
    ON td.pu_location_id = tz.location_id
JOIN silver.calendar cal
    ON cal.date = CONVERT(DATE, td.lpep_pickup_datetime)
JOIN silver.payment_type pt
    ON td.payment_type = pt.payment_type
WHERE td.year = '2020'
  AND td.month = '01'
GROUP BY
    td.year,
    td.month,
    tz.borough,
    CONVERT(DATE, td.lpep_pickup_datetime),
    cal.day_name;
```

#### Business Requirement 2: Taxi Demand Analysis

The business wants to understand taxi demand patterns across boroughs, days, weekends, and trip types.

The analysis includes:

1. Overall taxi demand.
2. Demand by borough.
3. Demand by weekday and weekend.
4. Demand by trip type, such as Street-hail and Dispatch.
5. Trip distance, trip duration, and fare amount by day and borough.

```sql
SELECT
    td.year,
    td.month,
    tz.borough,
    CONVERT(DATE, td.lpep_pickup_datetime) AS trip_date,
    cal.day_name AS trip_day,
    CASE WHEN cal.day_name IN ('Saturday', 'Sunday') THEN 'Y' ELSE 'N' END AS trip_day_weekend_ind,
    SUM(CASE WHEN pt.description = 'Credit card' THEN 1 ELSE 0 END) AS card_trip_count,
    SUM(CASE WHEN pt.description = 'Cash' THEN 1 ELSE 0 END) AS cash_trip_count,
    SUM(CASE WHEN tt.trip_type_desc = 'Street-hail' THEN 1 ELSE 0 END) AS street_hail_trip_count,
    SUM(CASE WHEN tt.trip_type_desc = 'Dispatch' THEN 1 ELSE 0 END) AS dispatch_trip_count,
    SUM(td.trip_distance) AS trip_distance,
    SUM(DATEDIFF(MINUTE, td.lpep_pickup_datetime, td.lpep_dropoff_datetime)) AS trip_duration,
    SUM(td.fare_amount) AS fare_amount
FROM silver.vw_trip_data_green td
JOIN silver.taxi_zone tz
    ON td.pu_location_id = tz.location_id
JOIN silver.calendar cal
    ON cal.date = CONVERT(DATE, td.lpep_pickup_datetime)
JOIN silver.payment_type pt
    ON td.payment_type = pt.payment_type
JOIN silver.trip_type tt
    ON td.trip_type = tt.trip_type
WHERE td.year = '2020'
  AND td.month = '01'
GROUP BY
    td.year,
    td.month,
    tz.borough,
    CONVERT(DATE, td.lpep_pickup_datetime),
    cal.day_name;
```

## 5. Reporting Requirements

The reporting layer focuses on delivering business-ready insights through Power BI.

Reporting areas:

- Taxi demand analysis.
- Credit card payment campaign analysis.
- Operational reporting.
- Borough, weekday, weekend, and trip type performance.
- Aggregated metrics for distance, duration, and fare amount.

### Sample Dashboard

![Power BI Sample Dashboard](https://github.com/AbdallahQoutbAli/Azure-Synapse-Analytics-For-Data-Engineers--Hands-On-Project/assets/47276503/cef39e63-83fc-462a-ae6d-0c1014de3ece)

### Dashboard Link

[NYC Taxi Trips Power BI Dashboard](https://app.powerbi.com/view?r=eyJrIjoiYzdkNWU1YzgtZGJjYi00Y2RlLTgyOTctMDA3NTRkNWM4MjRlIiwidCI6ImUwYjlhZTFlLWViMjYtNDZhOC1hZGYyLWQ3ZGJjZjIzNDBhOSJ9)

## 6. Key Skills Demonstrated

- Azure Synapse Analytics
- Serverless SQL Pool
- Data discovery using T-SQL
- External data sources and external file formats
- Data virtualization
- OPENROWSET over CSV, JSON, TSV, and Parquet files
- Partitioned data ingestion
- Views and stored procedures
- CETAS
- Data transformation using T-SQL
- Parquet-based analytical datasets
- Power BI reporting

## Author

**Abdallah Ali**  
BI Developer  
[LinkedIn](https://www.linkedin.com/in/abdallah-qoutb/)  
Abdallah.Qoutb@gmail.com
