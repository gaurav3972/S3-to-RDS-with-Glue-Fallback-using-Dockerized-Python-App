# 📦 **Automated Data Ingestion from Amazon S3 to RDS with AWS Glue Fallback (Dockerized Python App)**
---
## 📖 **Objective**

This project demonstrates a **fault-tolerant, Dockerized data ingestion pipeline** using AWS services. The application:

* Ingests a CSV file from **Amazon S3**
* Loads the data into **Amazon RDS (MySQL-compatible)**
* Falls back to **AWS Glue Data Catalog** if RDS upload fails
* Runs as a **containerized Python application on EC2 (Amazon Linux 2023)**

---

## 🚀 **Technologies Used**

| Category             | Technologies                               |
| -------------------- | ------------------------------------------ |
| **Cloud Services**   | AWS S3, RDS (MySQL), Glue, EC2             |
| **Language**         | Python 3.9                                 |
| **Libraries**        | `pandas`, `boto3`, `sqlalchemy`, `pymysql` |
| **Containerization** | Docker                                     |

---

# ✅ **PHASE 1: EC2 Instance Setup**

### 1.1 Launch EC2

* **AMI:** Amazon Linux 2023
* **Instance Type:** `t2.micro` (Free Tier)
* **Security Group:** Allow inbound traffic on ports:

  * `22` (SSH)
  * `3306` (MySQL)
* **Key Pair:** Download and save `.pem` file

---
![](https://github.com/gaurav3972/S3-to-RDS-with-Glue-Fallback-using-Dockerized-Python-App/blob/main/images/Screenshot%202025-07-14%20161136.png)
### 1.2 Connect to EC2

```bash
ssh -i your-key.pem ec2-user@<public-ip>
```

---

### 1.3 Install Docker

```bash
sudo dnf update -y
sudo dnf install docker -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user
exit  # Reconnect after adding docker group
```

Reconnect:

```bash
ssh -i your-key.pem ec2-user@<public-ip>
```

---

### 1.4 Verify Docker Installation

```bash
docker version
```

---

# ✅ **PHASE 2: Application Setup**

### 2.1 Create Project Structure

```bash
mkdir s3-to-rds-glue && cd s3-to-rds-glue
touch ingest.py requirements.txt Dockerfile
```

---

### 2.2 Python Script – `ingest.py`

```python
import os
import boto3
import pandas as pd
from sqlalchemy import create_engine
import pymysql

def read_csv_from_s3():
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=os.environ['S3_BUCKET'], Key=os.environ['S3_KEY'])
    return pd.read_csv(response['Body'])

def upload_to_rds(df):
    try:
        engine = create_engine(
            f"mysql+pymysql://{os.environ['RDS_USER']}:{os.environ['RDS_PASS']}@{os.environ['RDS_HOST']}/{os.environ['RDS_DB']}"
        )
        df.to_sql(os.environ['RDS_TABLE'], engine, index=False, if_exists='replace')
        print("✅ Uploaded to RDS")
        return True
    except Exception as e:
        print("❌ RDS failed:", e)
        return False

def fallback_to_glue():
    glue = boto3.client('glue')
    try:
        glue.create_database(DatabaseInput={'Name': os.environ['GLUE_DB']})
    except glue.exceptions.AlreadyExistsException:
        pass
    try:
        glue.create_table(
            DatabaseName=os.environ['GLUE_DB'],
            TableInput={
                'Name': os.environ['GLUE_TABLE'],
                'StorageDescriptor': {
                    'Columns': [{'Name': 'col1', 'Type': 'string'}, {'Name': 'col2', 'Type': 'string'}],
                    'Location': os.environ['GLUE_S3_LOCATION'],
                    'InputFormat': 'org.apache.hadoop.mapred.TextInputFormat',
                    'OutputFormat': 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat',
                    'SerdeInfo': {
                        'SerializationLibrary': 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe',
                        'Parameters': {'field.delim': ','}
                    }
                },
                'TableType': 'EXTERNAL_TABLE'
            }
        )
        print("✅ Glue table created")
    except glue.exceptions.AlreadyExistsException:
        print("⚠️ Glue table already exists")

if __name__ == "__main__":
    df = read_csv_from_s3()
    if not upload_to_rds(df):
        fallback_to_glue()
```

---

### 2.3 Python Dependencies – `requirements.txt`

```
boto3
pandas
sqlalchemy
pymysql
```

---

### 2.4 Dockerfile

```Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY ingest.py .

CMD ["python", "ingest.py"]
```

---

# ✅ **PHASE 3: Build & Run**

### 3.1 Build Docker Image

```bash
docker build -t s3-to-rds-glue-app .
```

---

### 3.2 Run Docker Container with Environment Variables

```bash
docker run --rm \
-e AWS_ACCESS_KEY_ID=your-access-key \
-e AWS_SECRET_ACCESS_KEY=your-secret-key \
-e AWS_REGION=us-east-1 \
-e S3_BUCKET=data-pipeline-gaurav \
-e S3_KEY=sample.csv \
-e RDS_HOST=mydb.rds.amazonaws.com \
-e RDS_USER=admin \
-e RDS_PASS=yourpassword \
-e RDS_DB=mydatabase \
-e RDS_TABLE=customer_data \
-e GLUE_DB=fallbackdb \
-e GLUE_TABLE=gaurav_table \
-e GLUE_S3_LOCATION=s3://data-pipeline-gaurav/glue-data/ \
s3-to-rds-glue-app
```
![](https://github.com/gaurav3972/S3-to-RDS-with-Glue-Fallback-using-Dockerized-Python-App/blob/main/images/Screenshot%202025-07-14%20161112.png)
---

# ✅ **PHASE 4: Validation**

### ✔️ Case 1: RDS Upload Succeeds

* **Log Output:** ![](https://github.com/gaurav3972/S3-to-RDS-with-Glue-Fallback-using-Dockerized-Python-App/blob/main/images/Screenshot%202025-07-14%20161219.png)
* **Verify:**

```sql
USE mydatabase;
SELECT * FROM customer_data;
```

---

### ❌ Case 2: RDS Upload Fails → Fallback to Glue
![](https://github.com/gaurav3972/S3-to-RDS-with-Glue-Fallback-using-Dockerized-Python-App/blob/main/images/Screenshot%202025-07-14%20162258.png)
* **Log Output:**

  * `❌ RDS failed: ...`
  * `✅ Glue table created` (or `⚠️ Glue table already exists`)
* **Verify via:** AWS Glue Console → Databases → Tables

---

# 📌 **Environment Variables Summary**

| Variable                | Purpose                           |
| ----------------------- | --------------------------------- |
| `AWS_ACCESS_KEY_ID`     | AWS access key                    |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key                    |
| `AWS_REGION`            | AWS region (e.g., `us-east-1`)    |
| `S3_BUCKET`             | Name of source S3 bucket          |
| `S3_KEY`                | CSV file key (path in bucket)     |
| `RDS_HOST`              | RDS endpoint (without `https://`) |
| `RDS_USER`              | RDS database user                 |
| `RDS_PASS`              | RDS password                      |
| `RDS_DB`                | RDS database name                 |
| `RDS_TABLE`             | Table name to write into          |
| `GLUE_DB`               | Glue database name                |
| `GLUE_TABLE`            | Glue table name                   |
| `GLUE_S3_LOCATION`      | S3 path used by Glue catalog      |

---

## 📌 **Tips & Notes**

* ✅ Ensure **IAM permissions** for accessing S3, RDS, and Glue.
* ✅ EC2 must be in the same **VPC/subnet** as RDS if RDS is not public.
* ✅ S3 file must be **CSV with valid headers** matching Glue column config.

---

## ✅ **Summary**

This project showcases a **robust and automated data pipeline** that:

* Pulls data from **Amazon S3**
* Inserts into **Amazon RDS (MySQL)**
* Fallbacks to **AWS Glue Data Catalog** in case of failure
* Is **containerized via Docker** and deployable on **EC2**

Ideal for real-world ETL and fault-tolerant ingestion workflows!

---

Let me know if you'd like to generate a PDF version or integrate this with Terraform for infrastructure automation.
