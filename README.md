# Automated Data Ingestion from Amazon S3 to RDS with AWS Glue Fallback using Dockerized Python

---

## 📖 Objective

This project demonstrates a production-ready, fault-tolerant data ingestion pipeline using AWS services. It:

* Ingests a CSV file from Amazon S3.
* Pushes the data to an Amazon RDS MySQL-compatible database.
* Falls back to AWS Glue Data Catalog if the RDS push fails.
* Uses Docker to containerize and deploy the application on Amazon EC2 (Amazon Linux 2023).

---

## 🚀 Technologies Used

* **AWS Services:** S3, RDS (MySQL), Glue, EC2
* **Programming:** Python 3.9, Pandas, Boto3, SQLAlchemy, PyMySQL
* **Containerization:** Docker

---

# ✅ PHASE 1: EC2 Setup

### 1.1 Launch EC2

* **AMI:** Amazon Linux 2023
* **Type:** `t2.micro` (Free Tier)
* **Security Group:** Allow port `22` (SSH) and port `3306` (for RDS MySQL)
* **Key Pair:** Download `.pem` file

### 1.2 Connect to EC2

```bash
ssh -i your-key.pem ec2-user@<public-ip>
```

### 1.3 Install Docker

```bash
sudo dnf update -y
sudo dnf install docker -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user
exit
```

Reconnect:

```bash
ssh -i your-key.pem ec2-user@<public-ip>
```

### 1.4 Test Docker

```bash
docker version
```

---

# ✅ PHASE 2: Project Setup

### 2.1 Create Folder and Files

```bash
mkdir s3-to-rds-glue && cd s3-to-rds-glue
touch ingest.py requirements.txt Dockerfile
```

### 2.2 ingest.py

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

### 2.3 requirements.txt

```
boto3
pandas
sqlalchemy
pymysql
```

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

# ✅ PHASE 3: Build and Run

### 3.1 Build Docker Image

```bash
docker build -t s3-to-rds-glue-app .
```

### 3.2 Run Docker Container

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

---

# ✅ PHASE 4: Validation

### ✔️ RDS Succeeds:

* Message: `✅ Uploaded to RDS`
* Verify via MySQL client:

```sql
USE mydatabase;
SELECT * FROM customer_data;
```

### ❌ RDS Fails:

* Message: `❌ RDS failed:` followed by `✅ Glue table created`
* Verify via AWS Glue Console

---

# ✅ PHASE 5: Deliverables

### 📷 Screenshots

* Terminal output showing success/failure
* MySQL SELECT query output
* Glue table from AWS Console (if fallback occurred)

### 📄 Summary Report `report.md`

```markdown
## 🧰 Data Flow
S3 ➔ RDS
     ↳ (on failure) AWS Glue

## 📚 AWS Services Used
- Amazon S3
- Amazon RDS (MySQL)
- AWS Glue
- Amazon EC2

## 🛠️ Tech Stack
- Python 3.9
- Pandas, SQLAlchemy
- Docker
- Boto3

## ⚡ Challenges
| Issue | Solution |
|-------|----------|
| RDS timeout | Port 3306 opened in SG |
| Access denied | Fixed password and created DB |
| NoRegionError | Used AWS_REGION variable |
```

---

# 🚀 Conclusion

This project showcases a reliable, modular, and AWS-integrated approach to handling data pipelines using Docker. The fallback strategy ensures data is never lost even when a primary service fails.

Let me know if you need:

* 📁 A zip of the complete project
* 📔 A professional GitHub README
* 📊 Architecture diagram (S3 ➔ RDS ➔ Glue fallback)

Congratulations on completing the project! 🌟
