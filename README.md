
# 📘 Automated Data Ingestion from Amazon S3 to RDS with AWS Glue Fallback (Dockerized Python App)
![](https://github.com/gaurav3972/S3-to-RDS-with-Glue-Fallback-using-Dockerized-Python-App/blob/main/images/0.0.0.png)
## 🎯 Objective

This project demonstrates how to build a fault-tolerant, automated data ingestion pipeline using:

- ✅ **Amazon S3** for data storage
- ✅ **Amazon RDS (MySQL)** as the primary database
- ✅ **AWS Glue** as a fallback mechanism if RDS insertion fails
- ✅ **Dockerized Python Application** for portability and consistent execution

---

## 🧰 Technologies Used

- **AWS S3** – Stores source CSV file  
- **AWS RDS (MySQL)** – Primary destination for data  
- **AWS Glue** – Backup if data cannot be inserted into RDS  
- **IAM** – Access control  
- **EC2** – Environment to run the Docker container  
- **Docker** – Containerization for consistent deployment  
- **Python Libraries** – `boto3`, `pandas`, `sqlalchemy`, `pymysql`

---

## 🛠️ Step-by-Step Implementation

### 🔹 Step 1: Launch EC2 Instance
1. Launch **Amazon Linux 2**.
2. Open these ports in the Security Group:
   - SSH (22)
   - HTTP (80) – optional
   - MySQL/Aurora (3306)
3. Install Docker:
```bash
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo usermod -aG docker ec2-user
````

---

### 🔹 Step 2: Setup IAM User

1. Go to IAM → Users → Create user
2. Username: `s3-rds-glue-user`
3. Enable **Programmatic Access**
4. Attach the following policies:

   * `AmazonS3FullAccess`
   * `AmazonRDSFullAccess`
   * `AWSGlueConsoleFullAccess`
5. Save the Access Key ID and Secret Access Key

---

### 🔹 Step 3: Create RDS (MySQL) Database

1. Go to RDS → Create Database
2. Config:

   * Engine: MySQL
   * DB Identifier: `rds-mysql`
   * DB Name: `mydb`
   * Username: `admin`
   * Password: `yourpassword`
   * Public access: Yes
   * Port: 3306
3. Add EC2 Security Group to the RDS inbound rules

#### ✅ Create Table:

```sql
CREATE DATABASE mydb;
USE mydb;
CREATE TABLE students (id INT, name VARCHAR(50));
```

---

### 🔹 Step 4: Upload CSV File to S3

1. Create a bucket (e.g., `my-data-bucket`)
2. Upload `data.csv`:

```csv
id,name
1,gaurav
2,patil
3,git
4,hub
5,project
```

---

### 🔹 Step 5: Python Script - `main.py`

A Python script that:

* Reads a CSV file from S3
* Inserts data into RDS
* Falls back to Glue if RDS insert fails

```python
# main.py
import boto3
import pandas as pd
import os
import pymysql
from sqlalchemy import create_engine

def read_csv_from_s3():
    ...

def upload_to_rds(df):
    ...

def fallback_to_glue():
    ...

if __name__ == "__main__":
    df = read_csv_from_s3()
    success = upload_to_rds(df)
    if not success:
        fallback_to_glue()
```

> Full script is included in this repo under `main.py`.

---

### 🔹 Step 6: Dockerfile

```dockerfile
FROM python:3.9
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
CMD ["python", "main.py"]
```

---

### 🔹 Step 7: `requirements.txt`

```text
boto3
pandas
sqlalchemy
pymysql
```

---

### 🔹 Step 8: `.env` File

```env
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_DEFAULT_REGION=ap-south-1

S3_BUCKET=kishor-data-ingest
CSV_KEY=data.csv

RDS_HOST=rds-endpoint.rds.amazonaws.com
RDS_PORT=3306
RDS_USER=admin
RDS_PASSWORD=yourpassword
RDS_DB=mydb
RDS_TABLE=students

GLUE_DATABASE=glue_fallback_db
GLUE_TABLE=students_glue
GLUE_S3_LOCATION=s3://kishor-data-ingest/
```

---

### 🔹 Step 9: Build Docker Image

```bash
docker build -t s3-rds-glue-app .
```

---

### 🔹 Step 10: Run the App

```bash
docker run --env-file .env s3-rds-glue-app
```

---

## 🔍 Output Scenarios

### ✅ Success:

```
📥 Reading CSV from S3...
✅ CSV loaded  
📤 Trying to upload data to RDS...
✅ Data uploaded to RDS
```

### 🔁 Fallback to Glue:

```
📥 Reading CSV from S3...
✅ CSV loaded  
📤 Trying to upload data to RDS...
❌ Upload to RDS failed: ...
🔁 Fallback to Glue triggered...
✅ Glue table created
```

---

## 📝 Project Summary

This project builds a **resilient and automated data pipeline** that:

* Pulls data from S3
* Pushes it to RDS (MySQL)
* Falls back to Glue on failure
* Is packaged as a Docker container for portability

---

