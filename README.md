# AWS Automated ETL Pipeline: S3 → Lambda → Glue

This repository contains the architecture and implementation of an event-driven data processing pipeline. The system automatically converts CSV files uploaded to Amazon S3 into JSON format using AWS Glue, with AWS Lambda acting as the orchestrator.

---

## 🏗 Architecture Overview

### Flow of Data & Control

1. **Ingestion:** A `CSV` file is uploaded to the `input_folder/` inside the Source S3 bucket.
2. **Trigger:** Amazon S3 detects the upload (`s3:ObjectCreated:*`) and sends an event notification to AWS Lambda.
3. **Orchestration:** Lambda extracts the bucket name and object key, then triggers the AWS Glue job.
4. **Transformation:** AWS Glue reads the CSV file, performs transformation using PySpark, and writes the JSON output to the Destination S3 bucket.

---

## 🚀 Setup Steps

### 1️⃣ Storage Configuration (Amazon S3)

- **Source Bucket:**  
  Create a bucket named `input-data-bucket`  
  Create a folder inside it: `input_folder/`

- **Destination Bucket:**  
  Create a bucket named `output-data-bucket`

- **Script Bucket:**  
  Create a bucket (e.g., `glue-internal-scripts`) to store the Glue ETL script.

---

### 2️⃣ IAM Roles & Permissions

#### 🔐 Glue Service Role
- Attach managed policy: `AWSGlueServiceRole`
- Add inline policy:
  - `s3:GetObject` → Source bucket
  - `s3:PutObject` → Destination bucket

#### 🔐 Lambda Execution Role
- Attach managed policy: `AWSLambdaBasicExecutionRole`
- Add inline policy:
  - `glue:StartJobRun` → Specific Glue job resource

---

### 3️⃣ AWS Glue ETL Job

- **Job Name:** `csv_to_json_etl_job`
- **Worker Type:** G.1X
- **Language:** PySpark
- **Script Location:** Stored in `glue-internal-scripts` bucket

The ETL script should:
- Read CSV from Source bucket
- Apply transformations (if required)
- Write output in JSON format to Destination bucket

---

### 4️⃣ AWS Lambda Function

- **Runtime:** Python 3.x
- **Purpose:** Trigger Glue job when a new file is uploaded to S3

```python
import boto3
import urllib.parse
import os

glue = boto3.client('glue')

def lambda_handler(event, context):
    # Get metadata from the S3 event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'])
    
    try:
        # Trigger the Glue Job
        response = glue.start_job_run(
            JobName='csv_to_json_etl_job',
            Arguments={
                '--SOURCE_BUCKET': bucket,
                '--SOURCE_KEY': key
            }
        )
        print(f"Triggered Glue Job for file: {key}")
        return response
    except Exception as e:
        print(f"Error triggering Glue job: {str(e)}")
        raise e
```


---
## 5️⃣ Configure S3 Trigger

1. Open the **AWS Lambda Console**.
2. Select your Lambda function.
3. Click **Add Trigger**.
4. Choose **S3** as the trigger source.
5. Select the bucket: `input-data-bucket`.
6. Set **Event Type** to: `All object create events`.
7. Set **Prefix** to: `input_folder/`.
8. Click **Add**.

This configuration ensures that whenever a new file is uploaded to the specified S3 prefix, the Lambda function is automatically triggered.

---

## 🧪 Testing the Pipeline

### 📤 Step 1: Upload Test File
Upload a sample CSV file to:`s3://input-data-bucket/input_folder/`

---

### 📊 Step 2: Monitor Lambda Execution
1. Open **Amazon CloudWatch**.
2. Navigate to **Logs**.
3. Select the log group for your Lambda function.
4. Confirm that the function was triggered successfully.

---

### 🔄 Step 3: Verify Glue Job Execution
1. Open the **AWS Glue Console**.
2. Go to **ETL Jobs** → **Job runs**.
3. Ensure the job run status shows **Succeeded**.

---

### ✅ Step 4: Validate Output
Check the output bucket:`s3://output-data-bucket/`

