### 1. Create an IAM Role for Lambda
1. Navigate to **IAM > Roles > Create Role**.
2. Select **AWS Service** > **Lambda**.
3. Attach the following policies:
   - `AmazonS3FullAccess`
   - `AWSLambdaBasicExecutionRole`
4. Name the role: `lambdaFunctionFullS3Access-CloudWatch`.

### 2. Create a Lambda Function to Copy Raw JSON Files
1. Navigate to **AWS Lambda > Create Function**.
2. Select **Author from Scratch**.
3. Enter function name: `copyRawJsonFile-lambdaFunction`.
4. Choose **Runtime**: Python 3.10.
5. Assign the existing role: `lambdaFunctionFullS3Access-CloudWatch`.
6. Click **Create Function**.
7. **Add Trigger:**
   - Select **S3**.
   - Choose **zillow-bucket**.
   - Select **All object create events**.
8. Replace `lambda_function.py` code with:

```python
import boto3
import json

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    object_key = event['Records'][0]['s3']['object']['key']

    target_bucket = 'copy-of-raw-json-bucket'
    copy_source = {'Bucket': source_bucket, 'Key': object_key}

    waiter = s3_client.get_waiter('object_exists')
    waiter.wait(Bucket=source_bucket, Key=object_key)
    s3_client.copy_object(Bucket=target_bucket, Key=object_key, CopySource=copy_source)

    return {
        'statusCode': 200,
        'body': json.dumps('Copy completed successfully')
    }
```
9. **Deploy the function**.

### 3. Create S3 Buckets
- `copy-of-raw-json-bucket`
- `cleaned-data-zone-csv-bucket`

### 4. Create a Lambda Function for Data Transformation
1. Navigate to **AWS Lambda > Create Function**.
2. Select **Author from Scratch**.
3. Enter function name: `transformation-convert-to-csv-lambdaFunction`.
4. Choose **Runtime**: Python 3.10.
5. Assign the existing role: `lambdaFunctionFullS3Access-CloudWatch`.
6. Click **Create Function**.
7. **Add Trigger:**
   - Select **S3**.
   - Choose **copy-of-raw-json-bucket**.
   - Select **All object create events**.
8. Replace `lambda_function.py` code with:

```python
import boto3
import json
import pandas as pd

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    object_key = event['Records'][0]['s3']['object']['key']
    
    target_bucket = 'cleaned-data-zone-csv-bucket'
    target_file_name = object_key[:-5]

    waiter = s3_client.get_waiter('object_exists')
    waiter.wait(Bucket=source_bucket, Key=object_key)

    response = s3_client.get_object(Bucket=source_bucket, Key=object_key)
    data = response['Body'].read().decode('utf-8')
    data = json.loads(data)
    
    df = pd.DataFrame(data['results'])
    selected_columns = ['bathrooms', 'bedrooms', 'city', 'homeStatus', 'homeType', 'livingArea', 'price', 'rentZestimate', 'zipcode']
    df = df[selected_columns]
    
    csv_data = df.to_csv(index=False)
    
    s3_client.put_object(Bucket=target_bucket, Key=f"{target_file_name}.csv", Body=csv_data)
    
    return {
        'statusCode': 200,
        'body': json.dumps('CSV conversion and S3 upload completed successfully')
    }
```
9. **Deploy the function**.

### 5. Configure Lambda Timeouts
1. Navigate to **Configuration > General Configuration**.
2. Edit timeout to **5 minutes** for both Lambda functions.

### 6. Add AWS SDK Layer to Transformation Function
1. Navigate to **AWS Lambda > transformation-convert-to-csv-lambdaFunction**.
2. Go to **Layers > Add a Layer**.
3. Select **AWS Layers**.
4. Choose `awssdkpandas-python310` (Version 4).

### 7. Set Up Airflow for Monitoring
1. **Create an Airflow DAG:**
   ```python
   from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

   is_file_in_s3_available = S3KeySensor(
       task_id='tsk_is_file_in_s3_available',
       bucket_key='{{ti.xcom_pull("tsk_extract_zillow_data_var")}}',
       bucket_name='s3_bucket',
       aws_conn_id='aws_s3_conn',
       wildcard_match=False,
       timeout=120,
       poke_interval=5,
   )
   
   extract_zillow_data_var >> load_to_s3 >> is_file_in_s3_available
   ```
2. **Run Airflow Standalone**.
3. **Create AWS Connection in Airflow:**
   - Go to **Admin > Connections**.
   - Click **+ New Connection**.
   - Set Connection ID: `aws_s3_conn`.
   - Connection Type: `Amazon Web Services`.
   - Add AWS Access and Secret Keys.

### 8. Run the DAG and Verify
- Trigger the DAG.
- Check S3 buckets to verify the copied and transformed files.

---
