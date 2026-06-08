# Snowflake + AWS S3 + Power BI End-to-End Analytics Lab
Project Overview
This project demonstrates the implementation of a complete analytics solution using AWS S3, Snowflake, and Power BI.

The objective was to simulate a modern enterprise analytics architecture, starting from raw CSV files stored in Amazon S3, loading and transforming data in Snowflake using a Medallion Architecture (Bronze, Silver, Gold), and finally delivering insights through a Power BI dashboard.

## Architecture
```
CSV Files
↓
AWS S3 Bucket
↓
Snowflake Storage Integration
↓
External Stage
↓
Bronze Layer (Raw Data)
↓
Silver Layer (Data Cleansing & Standardization)
↓
Gold Layer (Dimensional Model / Star Schema)
↓
Power BI Dashboard
```
## Technologies Used
Snowflake
AWS S3
SQL
Power BI
Data Warehousing
Dimensional Modeling
Star Schema Design
ETL / ELT Concepts

## Data Architecture
## Bronze Layer

Raw data loaded directly from CSV files stored in AWS S3.

Table:

TRAIN_RAW
## Silver Layer

Data cleansing and standardization.

Transformations included:

Date conversions
Data type standardization
Data quality validation

Table:

SALES_CLEAN
## Gold Layer

Business-ready dimensional model.

Dimensions:
```
DIM_CUSTOMER
DIM_PRODUCT
DIM_LOCATION
DIM_DATE
```
Fact Table:

FACT_SALES

The model uses surrogate keys to support dimensional modeling best practices.

## Power BI Data Model

A Star Schema was implemented using:
```
Customer Dimension
Product Dimension
Location Dimension
Date Dimension
Sales Fact Table
```
This design improves scalability, reporting performance, and maintainability.

## Key Learnings
Configuring Snowflake Storage Integrations
Connecting AWS S3 with Snowflake
Loading data using External Stages and COPY INTO
Implementing a Medallion Architecture
Building a dimensional model using surrogate keys
Developing a Power BI semantic model
Creating end-to-end analytics solutions
Dashboard

## The Power BI dashboard includes:
Total Sales
Total Orders
Average Order Value
Sales by Category
Sales by Region
Customer Analysis
Time-based Sales Trends
Repository Structure

## Data Lineage
Source:
AWS S3 Bucket

Raw Layer:
BRONZE.TRAIN_RAW

Transformation Layer:
SILVER.SALES_CLEAN

Presentation Layer:
GOLD.DIM_CUSTOMER
GOLD.DIM_PRODUCT
GOLD.DIM_LOCATION
GOLD.DIM_DATE
GOLD.FACT_SALES

Consumption Layer:
Power BI Dashboard

## sql/Snowflake
```
Database setup
create or replace warehouse lab_wh
with
warehouse_size = 'xsmall'
auto_suspend = 60
auto_resume = true
initially_suspended = true;

show warehouses;

use warehouse lab_wh;

create or replace database saleslab;

use database saleslab;

create or replace schema bronze;
create or replace schema silver;
create or replace schema gold;
```

## AWS integration scripts
```
create or replace storage integration S3_INT_LAB
TYPE = EXTERNAL_STAGE
STORAGE_PROVIDER = S3
ENABLED = TRUE
STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::008714537187:role/SNOWFLAKES3ROLE'
STORAGE_ALLOWED_LOCATIONS = ('s3://luis-bi-lab-2026/Lab1/');

SELECT CURRENT_VERSION();

DESC INTEGRATION S3_INT_LAB;
```


## Bronze layer scripts
```
CREATE OR REPLACE STAGE BRONZE.LAB1_STAGE
STORAGE_INTEGRATION = S3_INT_LAB
URL = 's3://luis-bi-lab-2026/Lab1/';

LIST @BRONZE.LAB1_STAGE;

DESC STAGE BRONZE.LAB1_STAGE;
SHOW STAGES IN SCHEMA BRONZE;

CREATE OR REPLACE FILE FORMAT BRONZE.CSV_FORMAT
TYPE = CSV
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"';

CREATE OR REPLACE TABLE BRONZE.TRAIN_RAW (
ROW_ID INTEGER,
ORDER_ID STRING,
ORDER_DATE STRING,
SHIP_DATE STRING,
SHIP_MODE STRING, 
CUSTOMER_ID STRING,
CUSTOMER_NAME STRING,
SEGMENT STRING,
COUNTRY STRING,
CITY STRING,
STATE STRING,
POSTAL_CODE STRING,
REGION STRING,
PRODUCT_ID STRING, 
CATEGORY STRING,
SUB_CATEGORY STRING,
PRODUCT_NAME STRING,
SALES NUMBER(18,2)
);

COPY INTO BRONZE.TRAIN_RAW
FROM @BRONZE.LAB1_STAGE/train.csv
FILE_FORMAT = BRONZE.CSV_FORMAT;

SHOW SCHEMAS;

SHOW SCHEMAS IN DATABASE SALESLAB;

SELECT COUNT (*)
FROM BRONZE.TRAIN_RAW;

SELECT *
FROM BRONZE.TRAIN_RAW
LIMIT 10;

## Data Validation
SELECT 
ORDER_DATE,
SHIP_DATE
FROM BRONZE.TRAIN_RAW
LIMIT 10;

SELECT ORDER_DATE
FROM BRONZE.TRAIN_RAW
WHERE ORDER_DATE LIKE '31/%'
LIMIT 5;
```

## Silver layer transformations
```
CREATE OR REPLACE TABLE SILVER.SALES_CLEAN AS 
SELECT
ROW_ID,
ORDER_ID,
TO_DATE(ORDER_DATE, 'DD/MM/YYYY') AS ORDER_DATE,
TO_DATE(SHIP_DATE, 'DD/MM/YYYY') AS SHIP_DATE,
SHIP_MODE,
CUSTOMER_ID,
CUSTOMER_NAME,
SEGMENT,
COUNTRY,
CITY,
STATE,
POSTAL_CODE,
REGION,
PRODUCT_ID,
CATEGORY,
SUB_CATEGORY,
PRODUCT_NAME,
SALES
FROM BRONZE.TRAIN_RAW;

DESC TABLE SILVER.SALES_CLEAN;

SELECT
MIN(ORDER_DATE),
MAX(ORDER_DATE)
FROM SILVER.SALES_CLEAN;
```
## Gold layer dimensional model
```
CREATE OR REPLACE TABLE GOLD.DIM_CUSTOMER AS
SELECT
ROW_NUMBER() OVER(
    ORDER BY CUSTOMER_ID) AS CUSTOMER_KEY,
CUSTOMER_ID,
CUSTOMER_NAME,
SEGMENT
FROM (
SELECT DISTINCT 
CUSTOMER_ID,
CUSTOMER_NAME,
SEGMENT
FROM
SILVER.SALES_CLEAN);

SELECT * 
FROM GOLD.DIM_CUSTOMER
LIMIT 10;


CREATE OR REPLACE TABLE GOLD.DIM_PRODUCT AS
SELECT 
ROW_NUMBER() OVER (
ORDER BY PRODUCT_ID) AS PRODUCT_KEY,
PRODUCT_ID,
PRODUCT_NAME,
CATEGORY,
SUB_CATEGORY
FROM ( 
SELECT DISTINCT
PRODUCT_ID,
PRODUCT_NAME,
CATEGORY,
SUB_CATEGORY
FROM SILVER.SALES_CLEAN);

CREATE OR REPLACE TABLE GOLD.DIM_LOCATION AS
SELECT
ROW_NUMBER () OVER(
ORDER BY COUNTRY, REGION, STATE, CITY) AS LOCATION_KEY,
COUNTRY,
REGION,
STATE,
CITY,
POSTAL_CODE
FROM (
SELECT DISTINCT
COUNTRY,
REGION,
STATE,
CITY,
POSTAL_CODE
FROM SILVER.SALES_CLEAN);

SELECT * 
FROM GOLD.DIM_LOCATION
LIMIT 10;
```

## Gold Joins
```
CREATE OR REPLACE TABLE GOLD.FACT_SALES AS
SELECT
S.ROW_ID,
S.ORDER_ID,
TO_NUMBER(
TO_CHAR(S.ORDER_DATE,'YYYYMMDD')
) AS ORDER_DATE_KEY,
TO_NUMBER(
TO_CHAR(S.SHIP_DATE,'YYYYMMDD')
)AS SHIP_DATE_KEY,

C.CUSTOMER_KEY,
P.PRODUCT_KEY,
L.LOCATION_KEY,

S.SALES
FROM SILVER.SALES_CLEAN S
INNER JOIN GOLD.DIM_CUSTOMER C
ON S.CUSTOMER_ID = C.CUSTOMER_ID
INNER JOIN GOLD.DIM_PRODUCT P
ON S.PRODUCT_ID = P.PRODUCT_ID
INNER JOIN GOLD.DIM_LOCATION L
ON S.COUNTRY = L.COUNTRY
AND S.REGION = L.REGION
AND S.STATE = L.STATE
AND S.CITY = L.CITY
AND S.POSTAL_CODE = L.POSTAL_CODE;

SELECT * 
FROM GOLD.FACT_SALES
LIMIT 10;

## DIM_Date
CREATE OR REPLACE TABLE GOLD.DIM_DATE AS
WITH DATE_SERIES AS (
SELECT DATEADD(
DAY,
SEQ4(),
'2015-01-01'::DATE) AS FULL_DATE
FROM TABLE(GENERATOR(ROWCOUNT =>2000))
)
SELECT 
TO_NUMBER(TO_CHAR(FULL_DATE, 'YYYYMMDD')) AS DATE_KEY,
FULL_DATE,
YEAR(FULL_DATE) AS YEAR,
QUARTER(FULL_DATE) AS QUARTER,
MONTH(FULL_DATE) AS MONTH,
MONTHNAME(FULL_DATE) AS MONTH_NAME,
DAY(FULL_DATE) AS DAY_OF_THE_MONTH,
DAYOFWEEK(FULL_DATE) AS DAY_OF_WEEK,
DAYNAME(FULL_DATE) AS DAY_NAME,
WEEK(FULL_DATE) AS WEEK_NUMBER
FROM DATE_SERIES;
```
## Some GOLD validations
```
SELECT *
FROM GOLD.DIM_DATE
LIMIT 10;

SELECT COUNT (*) FROM GOLD.FACT_SALES;
SELECT COUNT (*) FROM GOLD.DIM_CUSTOMER;
SELECT COUNT (*) FROM GOLD.DIM_PRODUCT;
SELECT COUNT (*) FROM GOLD.DIM_LOCATION;
SELECT COUNT (*) FROM GOLD.DIM_DATE;

SELECT CURRENT_ACCOUNT();
SHOW WAREHOUSES;

SELECT CURRENT_ORGANIZATION_NAME();
SELECT CURRENT_ACCOUNT_NAME();
SELECT CURRENT_REGION();
SELECT SYSTEM$ALLOWLIST();
SELECT CURRENT_USER();
```
/images

## Architecture diagrams
```
CSV Files
    ↓
AWS S3
    ↓
Snowflake Storage Integration
    ↓
External Stage
    ↓
BRONZE.TRAIN_RAW
    ↓
SILVER.SALES_CLEAN
    ↓
GOLD Star Schema
    ├── DIM_CUSTOMER
    ├── DIM_PRODUCT
    ├── DIM_LOCATION
    ├── DIM_DATE
    └── FACT_SALES
    ↓
Power BI Dashboard
```
## Data model screenshots
<img width="1342" height="621" alt="image" src="https://github.com/user-attachments/assets/943fc3b2-c476-4e1d-9b19-45f56d2590f1" />


## Dashboard screenshots
<img width="1152" height="647" alt="image" src="https://github.com/user-attachments/assets/67bbc76a-a892-4dee-bb80-eea1c58f3058" />

<img width="1150" height="647" alt="image" src="https://github.com/user-attachments/assets/f5fdabf7-4c39-459e-97ef-d7ccc4e8781f" />

<img width="1153" height="647" alt="image" src="https://github.com/user-attachments/assets/3679d1b7-02db-4037-a80e-24a0c87d4242" />


## powerbi

Power BI report file
https://app.powerbi.com/view?r=eyJrIjoiNjk1NzNhNjYtY2EyMC00M2YwLTgwMGEtMjFjZDVjN2VkNzVjIiwidCI6IjYyODliZTc5LTczOTktNDdlYi04Y2VkLTdlOWUzMzZmMjI2NCIsImMiOjZ9

## Author
Luis Méndez

Business Intelligence Manager | Data Analytics | Power BI | SQL | Snowflake
