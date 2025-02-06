### 1. **Create Redshift Cluster**
   - Log into the AWS Management Console and navigate to **Amazon Redshift**.
   - Click on **Create Cluster**.
   - Set the following parameters:
     - **Cluster Identifier**: `redshift-cluster-1`
     - **Node Type**: `ra3.xplus`
     - **Number of Nodes**: `1`
     - **Master Username**: `awsuserzillow`
     - **Master Password**: [Create a password]
   - Ensure the option **Publicly Accessible** is enabled in the cluster settings.
   - Click **Create Cluster** and wait for the cluster creation to complete (it may take some time).

### 2. **Connect to Redshift using Query Editor V2**
   - Go to the **Query Editor V2** section in the AWS Redshift Console.
   - Connect to `redshift-cluster-1` using the credentials created previously (`awsuserzillow` and the password).
   - Run the following SQL script to create the `zillowdata` table if it does not exist:

   ```sql
   CREATE TABLE IF NOT EXISTS zillowdata(
     bathrooms NUMERIC,
     bedrooms NUMERIC,
     city VARCHAR(255),
     homeStatus VARCHAR(255),
     homeType VARCHAR(255),
     livingArea NUMERIC,
     price NUMERIC,
     rentZestimate NUMERIC,
     zipcode INT
   );
   ```

   - Once the table is created, run the following query to check the data:

   ```sql
   SELECT * FROM zillowdata;
   ```

### 3. **Airflow Setup - Transfer Data from S3 to Redshift**

   - Create a Python script (`.py` file) in Airflow to transfer data from S3 to Redshift using the `S3ToRedshiftOperator`:

   ```python
   from airflow.providers.amazon.aws.transfer.s3_to_redshift import S3ToRedshiftOperator

   transfer_s3_to_redshift = S3ToRedShiftOperator(
       task_id='tsk_transfer_s3_to_redshift',
       aws_conn_id='aws_s3_conn',
       redshift_conn_id='conn_id_redshift',
       s3_bucket=s3_bucket,
       s3_key='{{ti.xcom_pull("tsk_extract_zillow_data_var")[1]}}',
       schema="PUBLIC",
       table="zillowdata",
       copy_options=["csv IGNOREHEADER 1"],
   )

   transfer_s3_to_redshift
   ```

### 4. **Add Connection in Airflow**
   - In Airflow, navigate to **Admin** > **Connections** and add a new connection:
     - **Connection ID**: `conn_id_redshift`
     - **Connection Type**: Amazon Redshift
     - **Host**: [Copy the Redshift Cluster endpoint (up to `.com`)]
     - **Database**: `dev`
     - **Username**: `awsuserzillow`
     - **Password**: [Your created password]
     - **Port**: `5439`
   - Save the connection.

### 5. **Add Permissions to EC2 Instance**
   - Go to **IAM** in the AWS Console.
   - Select the EC2 instance `ec2severalAcess-08232023`.
   - Attach the **AmazonRedshiftFullAccess** policy to the instance.

### 6. **Run the Airflow DAG**
   - In Airflow, navigate to your DAG and click **Trigger DAG**.
   - Go to the **Graph View** to monitor the tasks. Note: It may take some time to complete successfully.

### 7. **Verify Data in Redshift**
   - Go back to Redshift and run the following query to verify that the data has been transferred:

   ```sql
   SELECT * FROM zillowdata;
   ```

   - If the query runs successfully, congratulations! Your pipeline is working.

### 8. **QuickSight Setup for Data Analytics and Visualization**
   - Navigate to **AWS QuickSight** and sign up for the **Standard Edition** (not the Enterprise one).
   - Set up your account details and create a new dataset.
   - Select **Redshift** as the data source and enter the Redshift connection details you set up earlier.
   - Now, you can perform data analytics and create visualizations in QuickSight.

---

### Congratulations! You have successfully set up your Zillow ETL pipeline with Redshift, Airflow, and QuickSight for data visualization.