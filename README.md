# Local Database

# Student: Cecilia Quinto

# Project Overview

Initially explored other database options, but found PostgreSQL to be the best fit for this project since it aligns with all requirements such as scalability, complex querying capabilities, and local deployment.

Created PostgreSQL database locally using DataGrip.

This project documents the creation and use of a local PostgreSQL database to manage, clean and analyze experimental data collected from three types of sensors:

- GC (Gas Chromatograhy)
- IR (Infrared)
- E-Nose(Electronic Nose)

The data was collected over two experimental sessions, called Run 1, and Run 2. Each sensor has its own dataset per run, resulting in 6 total tables per ID. The primary goal is to prepare the data for statistical analysis.

## 🗂️ Folder Structure

```

project/
│
├── raw_data/                     ← Original CSV files from experiments
│   ├── carbon_e_nose/           ← E-nose sensor data
│   │   ├── 001run1.csv
│   │   ├── 001run2.csv
│   │   ├── 002run1.csv
│   │   └── ...
│   ├── defiant_gc/              ← Gas Chromatography (GC) data
│   │   ├── run1.csv
│   │   ├── run2.csv
│   │   └── ...
│   └── ring_ir/                 ← Infrared (IR) sensor data
│       ├── unm_001_1_2024.csv
│       ├── unm_001_2_2024.csv
│       ├── unm_002_2_2024.csv
│       └── ...
│
├── cleaned_data/                ← Cleaned CSVs for analysis
│   ├── run1_gc.csv              ← Contains all TEST_IDs for Run 1 using the GC sensor only
│   ├── run2_gc.csv              ← Contains all TEST_IDs for Run 2 using the GC sensor only
│   ├── run1_ir.csv              ← Contains all TEST_IDs for Run 1 using the IR sensor only
│   ├── run2_ir.csv              ← Contains all TEST_IDs for Run 2 using the IR sensor only
│   ├── run1_enose.csv           ← Contains all TEST_IDs for Run 1 using the E-nose sensor only
│   └── run2_enose.csv           ← Contains all TEST_IDs for Run 2 using the E-nose sensor only
│
├── sql/
│   ├── CreationTables.sql       ← SQL file to create all tables in PostgreSQL
│   ├── load_data.sql            ← SQL commands to import each cleaned CSV
│   └── Query.sql                ← SQL queries for data analysis and exploration
│
└── readme/
    └── README.md                ← Project documentation

---
# ✅ Step 1: Understanding the Raw Data

Each sensor produced a CSV file for each test ID and run. These files originally included:

- Time-series sensor readings for each Test ID (typically from 1 to 50)  
- Each run was recorded separately (e.g., Run 1 and Run 2)  
- Filenames contained the Test ID and run number in the file name, but not inside the CSV content, especially for E-nose and IR sensors.
- **E-nose example:** `001run1.csv`, `001run2.csv`, `002run1.csv`  
- **IR example:** `unm_001_1_2024.csv`, `uni_001_2.csv`, `unm_012_2_2024.csv`

Common issues found in the raw data:

- Missing or invalid values in some columns  
- Decimal values using commas instead of dots (e.g., `3,14` instead of `3.14`)  
- Empty cells or NaN values  
- Missing or mismatched column headers  
- Extra columns or misaligned rows in some files  
- Some files had no `TEST_ID` value inside the file; it was only found in the filename  
  - ➤ For consistency, we added a new `TEST_ID` column during preprocessing
 ---

## ✅ Step 2: Cleaning the Data

All CSV files were cleaned using a spreadsheet tool (Excel) and Python (Pandas library). The cleaning involved:

- **Assigning missing IDs**: Ensuring each row is correctly associated with a Test ID from 1 to 50.  
- **Correcting decimal formats**: Replacing commas (`,`) with dots (`.`) in numeric values to allow correct parsing.  
- **Filling missing values**: Instead of removing rows with missing data, appropriate placeholder values (e.g., NULL) were assigned to empty cells to preserve the structure and completeness of the dataset.
- **Standardizing columns**: Ensuring all files have the same structure: `test_id`, `timestamp`, and sensor readings.  
- **Saving cleaned files**: Exported as `.csv` again in UTF-8 encoding, ready to be imported into the database.  

---

## 🛠️ Step 3: Creating the Local PostgreSQL Database

We used **DataGrip** to connect to PostgreSQL, but any client (pgAdmin, DBeaver, CLI) will work.

### Database setup:

Create a new database called `sensor_data`:

```sql
CREATE DATABASE sensor_data;
```

Create tables using this example structure (`sql/CreationTables.sql`):

```sql
CREATE TABLE run1_gc (
    id SERIAL PRIMARY KEY,
    test_id INT,
    timestamp TIMESTAMP,
    sensor_value_1 FLOAT,
    sensor_value_2 FLOAT,
    ...
);
-- Repeat for run1_ir, run1_enorse, run2_gc, etc.
```

You will have 6 tables in total per ID:

run1_gc, run1_ir, run1_enorse
run2_gc, run2_ir, run2_enorse

---

## 📥 Step 4: Importing Cleaned Data into PostgreSQL

Using SQL’s `COPY` command to load each cleaned file:

```sql
COPY run1_gc(test_id, timestamp, sensor_value_1, sensor_value_2, ...)
FROM '/path/to/run1_gc_cleaned.csv'
DELIMITER ','
CSV HEADER;
```

✅ Make sure PostgreSQL has permission to access that file path. You may also use **DataGrip’s import tool** (`right-click table → Import from CSV`).  
Repeat for **all six cleaned CSVs**.

---
## 📊 Step 5: Querying the Data

Once data is in the database, you can begin basic exploration and comparison between sensors. Example queries:

## 1. Query: Combine All Tables from Both Runs

```sql
SELECT *
FROM r1_ir
LEFT JOIN r1_gc ON CAST(r1_ir.testid AS VARCHAR) = r1_gc.testid
LEFT JOIN r1_e_norse ON r1_gc.testid = r1_e_norse.testid

UNION ALL

SELECT *
FROM r2_ir
LEFT JOIN r2_gc ON CAST(r2_ir.testid AS VARCHAR) = r2_gc.testid
LEFT JOIN r2_e_norse ON r2_gc.testid = r2_e_norse.testid;
```

Purpose:
Join the infrared (ir), gas chromatography (gc), and e-norse sensor tables for run 1 and run 2 into a single result set.
Note: Casting is used for type compatibility on join keys.

## 2. Query: Join Run 1 Tables

```sql
SELECT *
FROM r1_ir
LEFT JOIN r1_gc ON r1_ir.testid = r1_gc.testid
LEFT JOIN r1_e_norse ON r1_gc.testid = r1_e_norse.testid;

```
Purpose:
Join the three sensor tables (ir, gc, e_norse) for run 1 to analyze combined data.

## 3. Query: Select Specific Columns from IR Table (Run 1)

```sql
SELECT ratios, harmonic, fundamental, location, testid
FROM r1_ir;

```

Purpose:
Retrieve specific relevant columns from the infrared sensor data for run 1.

## 4. Query: Filter GC Table (Run 1) by Test ID

```sql
SELECT *
FROM r1_gc
WHERE testid = 1;
```

Purpose:
Select all columns for test ID 1 from the gas chromatography data in run 1.

## 5. Query: Select All Data from E-Norse Table (Run 1), Ordered by Test ID and Sequence

```sql
SELECT *
FROM r1_e_norse
ORDER BY testid, seq_order ASC;
```

Purpose:
Retrieve all e-norse sensor data for run 1, sorted by test ID and sequence order.

## 6. Query: Verify Number of Distinct Test IDs in Each Table

```sql
SELECT COUNT(DISTINCT testid) FROM r1_gc;
SELECT COUNT(DISTINCT testid) FROM r1_ir;
SELECT COUNT(DISTINCT testid) FROM r1_e_norse;
SELECT COUNT(DISTINCT testid) FROM r2_gc;
SELECT COUNT(DISTINCT testid) FROM r2_ir;
SELECT COUNT(DISTINCT testid) FROM r2_e_norse;
```

Purpose:
Count how many unique tests (testid) are present in each sensor table for both experimental runs. Useful for data completeness checks.

## 7. Query: Calculate Average E-Norse Sensor Value per Test (Run 1)

```sql
SELECT
    testid,
    AVG((
        D1 + D2 + D3 + D4 + D5 + D6 + D7 + D8 + D9 + D10 +
        D11 + D12 + D13 + D14 + D15 + D16 + D17 + D18 + D19 + D20 +
        D21 + D22 + D23 + D24 + D25 + D26 + D27 + D28 + D29 + D30 +
        D31 + D32 + D33 + D34 + D35 + D36 + D37 + D38 + D39 + D40 +
        D41 + D42 + D43 + D44 + D45 + D46 + D47 + D48 + D49 + D50 +
        D51 + D52 + D53 + D54 + D55 + D56 + D57 + D58 + D59 + D60 +
        D61 + D62 + D63 + D64
    ) / 64.0) AS avg_sensor_val
FROM r1_e_norse
GROUP BY testid;
```

Purpose:
Compute the average sensor reading across all 64 individual e-norse sensors for each test in run 1.

## 8. Query: Calculate Average Humidity and Temperature per Test (Run 1)

```sql
SELECT testid,
       AVG(Humidity) AS avg_humidity,
       AVG(Temperature) AS avg_temp
FROM r1_e_norse
GROUP BY testid;
```
Purpose:
Calculate average environmental conditions (humidity and temperature) for each test in run 1.

## 9. Query: Calculate Average Reading per Individual E-Norse Sensor (Run 1)

```sql
SELECT
    AVG(D1) AS avg_d1,
    AVG(D2) AS avg_d2,
    -- ... all sensors through D64 ...
    AVG(D64) AS avg_d64
FROM r1_e_norse;

```

Purpose:
Assess which individual e-norse sensors tend to have higher or lower readings on average.

## 10. Query: Find Maximum and Minimum Sensor Values per Test ID (Run 1)

```sql
SELECT
    testid,
    MAX(D1) AS max_d1, MIN(D1) AS min_d1,
    MAX(D2) AS max_d2, MIN(D2) AS min_d2,
    -- ... all sensors through D50 ...
    MAX(D50) AS max_d50, MIN(D50) AS min_d50
FROM r1_e_norse
GROUP BY testid;

```
Purpose:
Identify the range of sensor values per test, which is useful for detecting anomalies or sensor behavior extremes.

## 11. Query: Filter E-Norse Data by Average Sensor Value Threshold (Run 1)

```sql
SELECT *
FROM r1_e_norse
WHERE (
    D1 + D2 + D3 + ... + D64
) / 64.0 > 0.8;
```

Purpose:
Select records where the average sensor reading (across all 64 sensors) is greater than 0.8, to focus on potentially significant data points.


# An-Introduction-to-DataGrip
**Important Tip:** Before opening any links (blue text), we highly recommend right-clicking on them and selecting **"Open link in new tab"** to ensure the guide remains open.

# DataGrip Installation Guide 🛢️
### What is PostgreSQL?
DataGrip is a powerful database IDE developed by JetBrains, designed for database developers and administrators. It supports multiple database management systems

## ✨ STEP 1 - Download 💾
- The first step is to go to the following link [JetBrains DataGrip](https://www.jetbrains.com/datagrip/) , which will redirect you to JetBrains' official website, where you can download DataGrip. Accept the cookies and proceed with the download
- Next, choose the appropriate download option based on your operating system (Windows, macOS, or Linux) and click "Download."
- ![image](https://github.com/user-attachments/assets/8ccdd232-e62e-48c6-ad6c-1291656ae0d2)

## ⚙️STEP 2 - Installation 🛠️ 
- After opening the downloaded file, the setup menu will appear. Click "Next", and when the Install Location screen appears, simply click "Next" again to proceed with the default folder.
- In the Installation Options screen, check the recommended options if preferred, although you can proceed without selecting any.
- ![image](https://github.com/user-attachments/assets/7d43d689-0acb-457b-89cf-a7afb8e0d1b1)
- In the Start Menu Folder screen, no changes are needed, so simply click "Install" to proceed.
- And that's it! 😊 Check the "Run DataGrip" option and click "Finish" to complete the installation.

## 🔐 STEP 3 - Activation 🛡️
- Once installed, a User Agreement window will appear. Simply click "Confirm and Continue" to proceed.
- Regarding Data Sharing, it is entirely up to you—it is not required.
- Now, the application will launch. Click on "Activate License", then go to the "Subscription" tab, where you will find the option to log in to JetBrains. Click on it to proceed.
- This will redirect you to Google or your default web browser. Important: Click on "Log in with Google" and use the account associated with your purchased license. Once logged in, you're all set! Just return to the app/IDE.
- Back in the app/IDE, click "Activate", and you're all set! Now, we can move on to the next part of the guide.

# PostgreSQL Installation Guide  🐘
### What is PostgreSQL?
PostgreSQL is a powerful, open-source relational database management system known for its reliability, extensibility, and compliance with SQL standards.

## STEP 1 - Download 💾
- The first step is to go to the following link [PostgreSQL](https://www.postgresql.org/download/). Once there, click on the option that matches your operating system.
-![image](https://github.com/user-attachments/assets/286141b0-d464-4708-bfd5-b5b38c896ab3)
- Next, click on "Download the installer." If you're unsure where it is, refer to the arrow in the following image.
- ![image](https://github.com/user-attachments/assets/46f27a71-c9c1-43b6-9e53-0d7b46fd264d)
- Then, click on the version 17.4 that matches your operating system, whether it's Windows or macOS.
- ![image](https://github.com/user-attachments/assets/86921748-0eb9-4b4e-aaa7-f7cb8ab60605)
- Next, the file will start downloading. Once it's finished, open it. Note: Registration on EDB is not required.

## ⚙️STEP 2 - Installation 🛠️ 
- Now, a setup window will appear. Click "Next." This setup process is a bit more complex than the previous one, so pay close attention.
- In the Installation Directory screen, simply click "Next."
- In the Select Components screen, only PostgreSQL Server and Command Line Tools are needed—select these and proceed.
- ![image](https://github.com/user-attachments/assets/85bf3d7b-af5e-4f57-bcb7-f1422ea555d6)
- In the Data Directory screen, click "Next."
- Now, in the Password section—this part is **very important**. Enter a password that **you will remember**, as we will need it frequently to access the database. If you're unsure, **write it down somewhere safe** or use another method to store it securely.
- In the Port section, leave it as 5432 and click "Next" to proceed.
- In the Advanced Options section, leave everything as it is and click "Next," just like in the Pre-Installation Summary screen.
- Once the installation is complete, click "Finish," and that's it! 😊

# DataGrip Custom 🎨 (OPTIONAL)
### Description
This is a small optional section of the guide that you can skip. It focuses on customization to make your experience with DataGrip more pleasant and user-friendly.

## Customize
- Theme: If you're like me and find bright screens a bit overwhelming, you can switch to Dark Mode, which is especially helpful for reducing eye strain at night.
- IDE Font: If you need larger text for better visibility when working with data or writing code, you can adjust the font size here.
- Region: You can set the region to Americas in the Region option, though this isn’t crucial. Keep in mind that changing this setting requires a restart.

## Plugins
To search for plugins, go to the search bar and type the names of the plugins you want to find.

- A must-have plugin is Atom Material Icons, which replaces file icons with their relevant logos, making navigation more intuitive.
- If you want more theme options, I highly recommend Material Theme UI, which offers a variety of stylish and customizable themes.

# First Steps in DataGrip 🎯
### Description
Now, we will proceed with setting up the databases on your computer, allowing you to visualize data, analyze it, and even generate charts. The chart generation part will be covered later in the guide.

## STEP 1 - Data BASE 🛢️
- We open DataGrip again and click on "Create New Project."
- We will name it "UNM - Database."
- Click on "Create Data Source", which is located on the left side.
- ![image](https://github.com/user-attachments/assets/8b41beec-b413-42dd-9745-9d2fa5d5aeb8)
- Then, select PostgreSQL, which is among the last options.
- A new window will open, and the first thing you need to do is click on "Download missing drivers", which is indicated at the bottom.
- Next, go to the User field and enter "postgres".
- In the Password field, enter the password you created earlier.
- And that's it! Now, click on "Test Connection" to verify that everything is working correctly. If you encounter an error, double-check the previous steps. If the issue persists, feel free to contact me for help. 😊
- ![image](https://github.com/user-attachments/assets/8d6468c3-924c-425f-8039-7062637d5190)
- Click "OK", and you're all set! 🎉

## STEP 2 - Tables 📊
- Now, double-click to open @localhost → postgres → public.
- Right-click on public, select New, and then click on Query Console.
- Now, we will create the IR Test table. We have separated it into two: R1 for Run 1 and R2 for Run 2.
- Now, copy and paste this code into the opened query console.
```sql
CREATE TABLE ir_test_r1 (
    ratio NUMERIC(20) ,
    funamental INTEGER,
    harmonic INTEGER,
    wave_leght VARCHAR,
    boxid VARCHAR,
    location VARCHAR,
    time_stamp VARCHAR,
    counter_id VARCHAR,
    es_ver VARCHAR,
    dp_ver VARCHAR,
    database_match VARCHAR,
    test_id INTEGER 
);
```
- Now, click on Run (the green play button) to execute the query.
- ![image](https://github.com/user-attachments/assets/ce0dc156-65a5-4d9f-8a10-3b53dbca74dd)
- If no errors appear in the console, we can continue. To verify, double-click on public, then go to Tables, and you should see the newly created table.
- Now that the table is created, delete the part that says "CREATE TABLE ir_test_r1" in the code and replace it with "CREATE TABLE ir_test_r2". This simply changes the table name to create the table for Run 2.
- Now, let's move on to the next table, E-Norse Repeat the previous steps by either creating a new Query Console or deleting the previous code (make sure you have already run the Run 2 code and verified that the table was created). Now, paste the following code:
```sql
CREATE TABLE e_Norse_r1 (
    test_id INTEGER,
    seq_order INTEGER,
    seq_name VARCHAR,
    seq_time INTEGER,
    pump_rate INTEGER,
    v1_pos INTEGER,
    v2_pos INTEGER,
    uv_led BOOLEAN,
    record_data BOOLEAN,
    result VARCHAR,
    date VARCHAR,
    D1 DOUBLE PRECISION,
    D2 DOUBLE PRECISION,
    D3 DOUBLE PRECISION,
    D4 DOUBLE PRECISION,
    D5 DOUBLE PRECISION,
    D6 DOUBLE PRECISION,
    D7 DOUBLE PRECISION,
    D8 DOUBLE PRECISION,
    D9 DOUBLE PRECISION,
    D10 DOUBLE PRECISION,
    D11 DOUBLE PRECISION,
    D12 DOUBLE PRECISION,
    D13 DOUBLE PRECISION,
    D14 DOUBLE PRECISION,
    D15 DOUBLE PRECISION,
    D16 DOUBLE PRECISION,
    D17 DOUBLE PRECISION,
    D18 DOUBLE PRECISION,
    D19 DOUBLE PRECISION,
    D20 DOUBLE PRECISION,
    D21 DOUBLE PRECISION,
    D22 DOUBLE PRECISION,
    D23 DOUBLE PRECISION,
    D24 DOUBLE PRECISION,
    D25 DOUBLE PRECISION,
    D26 DOUBLE PRECISION,
    D27 DOUBLE PRECISION,
    D28 DOUBLE PRECISION,
    D29 DOUBLE PRECISION,
    D30 DOUBLE PRECISION,
    D31 DOUBLE PRECISION,
    D32 DOUBLE PRECISION,
    D33 DOUBLE PRECISION,
    D34 DOUBLE PRECISION,
    D35 DOUBLE PRECISION,
    D36 DOUBLE PRECISION,
    D37 DOUBLE PRECISION,
    D38 DOUBLE PRECISION,
    D39 DOUBLE PRECISION,
    D40 DOUBLE PRECISION,
    D41 DOUBLE PRECISION,
    D42 DOUBLE PRECISION,
    D43 DOUBLE PRECISION,
    D44 DOUBLE PRECISION,
    D45 DOUBLE PRECISION,
    D46 DOUBLE PRECISION,
    D47 DOUBLE PRECISION,
    D48 DOUBLE PRECISION,
    D49 DOUBLE PRECISION,
    D50 DOUBLE PRECISION,
    D51 DOUBLE PRECISION,
    D52 DOUBLE PRECISION,
    D53 DOUBLE PRECISION,
    D54 DOUBLE PRECISION,
    D55 DOUBLE PRECISION,
    D56 DOUBLE PRECISION,
    D57 DOUBLE PRECISION,
    D58 DOUBLE PRECISION,
    D59 DOUBLE PRECISION,
    D60 DOUBLE PRECISION,
    D61 DOUBLE PRECISION,
    D62 DOUBLE PRECISION,
    D63 DOUBLE PRECISION,
    D64 DOUBLE PRECISION,
    Humidity DOUBLE PRECISION,
    Temperature DOUBLE PRECISION,
    "D-Norm1" DOUBLE PRECISION,
    "D-Norm2" DOUBLE PRECISION,
    "D-Norm3" DOUBLE PRECISION,
    "D-Norm4" DOUBLE PRECISION,
    "D-Norm5" DOUBLE PRECISION,
    "D-Norm6" DOUBLE PRECISION,
    "D-Norm7" DOUBLE PRECISION,
    "D-Norm8" DOUBLE PRECISION,
    "D-Norm9" DOUBLE PRECISION,
    "D-Norm10" DOUBLE PRECISION,
    "D-Norm11" DOUBLE PRECISION,
    "D-Norm12" DOUBLE PRECISION,
    "D-Norm13" DOUBLE PRECISION,
    "D-Norm14" DOUBLE PRECISION,
    "D-Norm15" DOUBLE PRECISION,
    "D-Norm16" DOUBLE PRECISION,
    "D-Norm17" DOUBLE PRECISION,
    "D-Norm18" DOUBLE PRECISION,
    "D-Norm19" DOUBLE PRECISION,
    "D-Norm20" DOUBLE PRECISION,
    "D-Norm21" DOUBLE PRECISION,
    "D-Norm22" DOUBLE PRECISION,
    "D-Norm23" DOUBLE PRECISION,
    "D-Norm24" DOUBLE PRECISION,
    "D-Norm25" DOUBLE PRECISION,
    "D-Norm26" DOUBLE PRECISION,
    "D-Norm27" DOUBLE PRECISION,
    "D-Norm28" DOUBLE PRECISION,
    "D-Norm29" DOUBLE PRECISION,
    "D-Norm30" DOUBLE PRECISION,
    "D-Norm31" DOUBLE PRECISION,
    "D-Norm32" DOUBLE PRECISION
);
```
- Repeat the previous step by changing "CREATE TABLE e_Norse_r1" to "CREATE TABLE e_Norse_r2" to create the table for Run 2.
