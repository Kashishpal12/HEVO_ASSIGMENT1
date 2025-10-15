# 🚀 Hevo Data Assessment – End-to-End Data Pipeline (PostgreSQL → Hevo → Snowflake)

This project demonstrates the setup of a **complete data pipeline** using **PostgreSQL (Neon)**, **Hevo Data**, and **Snowflake**.  
It covers every step from environment setup, data ingestion, transformation (via Python), and validation in the destination warehouse.

---

## 📖 Overview

The goal of this project was to:

- ✅ Set up a **Snowflake trial account** and connect it with a **Hevo partner account**
- ✅ Establish a working **PostgreSQL** environment
- ✅ Ingest data from CSV files into PostgreSQL
- ✅ Connect PostgreSQL to Hevo as a source
- ✅ Apply transformations using **Python** in Hevo
- ✅ Load transformed data into **Snowflake**
- ✅ Validate the final results using SQL queries

---

## 🏁 Step 1: Destination Setup (Snowflake Partner Connect)

### 1️⃣ Snowflake Trial Account Creation
- Created a Snowflake trial account using Gmail.
- Logged in as admin.

### 2️⃣ Hevo Partner Account Setup
- Inside Snowflake, navigated to **Admin → Partner Connect**.
- Selected **Hevo Data** and authorized connection.
- Hevo workspace auto-provisioned.

✅ *Hevo connected to Snowflake successfully.*

---

## 🐳 Step 2: Initial Setup Attempt with Docker (and Why It Was Replaced)

### 1️⃣ Docker Setup
- Installed Docker Desktop and attempted to create a local PostgreSQL container.
- Verified installation using:
  ```bash
  docker version
  ```
  ✅ Command worked successfully.

### 2️⃣ Issue Encountered
- Launching the container threw repeated **500 API errors**.

### 3️⃣ Decision
- Switched to **Neon.tech**, a managed cloud PostgreSQL service with:
  - Free tier  
  - GUI  
  - Logical replication support

---

## 🧱 Step 3: PostgreSQL Setup in Neon

- Signed up at [Neon Console](https://console.neon.tech).
- Created project and database **`neondb`**.
- Default role: `neondb_owner`.

✅ *Connection successful.*

### Connection via pgAdmin
Created a new server named **neondb** using these credentials:

| Parameter | Value |
|------------|--------|
| Hostname | `ep-delicate-math-adkswflv-pooler.c-2.us-east-1.aws.neon.tech` |
| Database | `neondb` |
| User | `neondb_owner` |
| Password | `********` |

✅ *pgAdmin connected successfully.*

---

## 🧮 Step 4: Table Creation and Data Import under pgAdmin

### 1️⃣ Table Creation
```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    address JSON
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT,
    status VARCHAR(50),
    total_amount NUMERIC(10,2)
);

CREATE TABLE feedback (
    order_id INT PRIMARY KEY,
    rating INT,
    comment TEXT
);
```

### 2️⃣ Data Import
Downloaded CSV files from:  
📂 [Hevo Assessment CSV Repo](https://github.com/muskan-kesharwani-hevo/hevo-assessment-csv)

- `customers.csv`
- `orders.csv`
- `feedback.csv`

Imported using pgAdmin’s **Import/Export Data** tool.

---

### ⚠️ Issue 1: Invalid JSON Format
**Error:**  
`invalid input syntax for type json`

**Root Cause:**  
`address` column values not enclosed in double quotes.

**Fix:**  
Converted column type from `JSON` → `TEXT`
```sql
ALTER TABLE customers ALTER COLUMN address TYPE TEXT USING address::TEXT;
```
✅ *Re-imported successfully.*

---

### ⚠️ Issue 2: Duplicate Key Violation
**Error:**  
```
ERROR: duplicate key value violates unique constraint "feedback_order_id_key"
DETAIL: Key (order_id)=(1121) already exists.
```

**Root Cause:**  
133 duplicate `order_id` values in `feedback.csv`.

**Fix using Python (Pandas):**
```python
import pandas as pd
df = pd.read_csv('feedback.csv')
df = df.drop_duplicates(subset='order_id', keep='first')
df.to_csv('feedback_clean.csv', index=False)
```
✅ *Re-imported `feedback_clean.csv` successfully.*

---

## 🔗 Step 5: Connect PostgreSQL to Hevo

### 1️⃣ Pipeline Setup
- Source: PostgreSQL (Neon)
- Destination: Snowflake

### 2️⃣ Connection Configuration
| Parameter | Value |
|------------|--------|
| Host | `ep-delicate-math-adkswflv-pooler.c-2.us-east-1.aws.neon.tech` |
| Database | `neondb` |
| User | `neondb_owner` |
| Password | `********` |

### 3️⃣ Error Encountered
Hevo error:
> Unable to use logical replication. wal_level must be set to 'logical'

### 4️⃣ Fix
- Logged in to **Neon → Settings → Enable Logical Replication**
- Verified using:
  ```sql
  SHOW wal_level;
  ```
  Output: `logical`

✅ *Reconnected successfully.*

---

## ⚙️ Step 6: Hevo Pipeline Configuration

| Setting | Value |
|----------|--------|
| Destination Table Prefix | *(Blank)* |
| Auto Mapping | Enabled |
| Ingestion Schedule | Every 30 minutes |

✅ *Tables from PostgreSQL successfully appeared in Hevo.*

---

## 🧠 Step 7: Data Transformation in Hevo (Python Script)

### Transformation Goals
1. Derive **username** from email in `customers`
2. Create new derived table **order_events** from `orders`
3. Pass other tables unchanged

### Python Script
```python
import datetime

def transform(event):
    table_name = event.get('__table_name')
    if not table_name:
        return event, None

    # 1️⃣ Add username for customers
    if table_name == 'customers':
        email = event.get('email')
        if email and '@' in email:
            event['username'] = email.split('@')[0]
        return event, table_name

    # 2️⃣ Generate order_events for orders
    elif table_name == 'orders':
        status_map = {
            'delivered': 'order_delivered',
            'placed': 'order_placed',
            'shipped': 'order_shipped',
            'cancelled': 'order_cancelled'
        }
        status = event.get('status')
        output_records = [(event, table_name)]

        if status in status_map:
            new_event_record = {
                'order_id': event.get('id'),
                'customer_id': event.get('customer_id'),
                'event_type': status_map[status],
                'event_timestamp': datetime.datetime.now().isoformat()
            }
            output_records.append((new_event_record, 'order_events'))
        return output_records

    # 3️⃣ Pass feedback and other tables
    return event, table_name
```

✅ *Transformation deployed successfully.*

---

## ❄️ Step 8: Load and Validate in Snowflake

**Database:** `PC_HEVODATA_DB`  
**Schema:** `PUBLIC`

### SQL Operations
```sql
ALTER TABLE PC_HEVODATA_DB.PUBLIC.CUSTOMERS
ADD COLUMN USERNAME VARCHAR;

UPDATE PC_HEVODATA_DB.PUBLIC.CUSTOMERS
SET USERNAME = SPLIT_PART(EMAIL, '@', 1);
```

---

### ✅ Validation Queries

**Validation A — Order Events**
```sql
SELECT EVENT_TYPE, COUNT(*)
FROM PC_HEVODATA_DB.PUBLIC.ORDER_EVENTS
GROUP BY EVENT_TYPE;
```

**Validation B — Username Field**
```sql
SELECT EMAIL, USERNAME, FIRST_NAME
FROM PC_HEVODATA_DB.PUBLIC.CUSTOMERS
WHERE USERNAME IS NOT NULL
LIMIT 10;
```

✅ *All validation checks passed successfully.*

---

## 💡 Issues Encountered & Design Choices

| Category | Decision / Assumption |
|-----------|------------------------|
| Address Field | Converted JSON → TEXT for simplicity |
| Order Status Type | Used VARCHAR instead of ENUM for flexibility |
| Duplicate Handling | Removed duplicates at source using Python |
| Transformation Type | Used Python for precise control |
| Ingestion Frequency | 30-minute schedule for steady sync |

---

## 📁 Repository Structure

```
📦 hevo-data-assessment
│
├── README.md
├── sql/
│   ├── create_tables.sql
│   ├── validation_queries.sql
│
├── data/
│   ├── customers.csv
│   ├── orders.csv
│   ├── feedback_clean.csv
│
├── transform/
│   └── transform_script.py
```

---

## 🧩 Summary

✅ Attempted PostgreSQL via Docker (failed due to 500 API error)  
✅ Switched to **Neon PostgreSQL** (successful)  
✅ Created & populated tables  
✅ Connected **PostgreSQL → Hevo** with logical replication  
✅ Applied **Python transformations**  
✅ Synced data to **Snowflake**  
✅ Verified transformations and data integrity  

---

### 🏷️ About

> **Author:** Kashish Pal  
> **Project:** Hevo Data Assessment I  
> **Technologies:** PostgreSQL (Neon), Hevo Data, Snowflake, Python, SQL  

---

📘 *This repository demonstrates an end-to-end ELT workflow leveraging cloud-native data engineering tools for modern analytics pipelines.*
