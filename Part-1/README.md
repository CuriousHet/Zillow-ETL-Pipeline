## 1. AWS User Setup

### Create User Group
- **User group**: `zillowgroup` (AdministratorAccess)

### Create User
- **User**: `dataguyhet`
- Add user to `zillowgroup`
- Create an **access key** for CLI access
- Log in using the `dataguyhet` user

---

## 2. Launch EC2 Instance

### Steps to Launch an Instance
- **Instance Name**: `zillow-EC2`
- **OS**: Ubuntu
- **Instance Type**: `T2.medium`
- **Keypair**: (Use existing or create a new one)
- **Security Group**: Allow all traffic
- **Launch Instance**
- **Connect to Instance**: 
  ```sh
  ssh -i <your-key.pem> ubuntu@<EC2-IP>
  ```

### Install Dependencies
```sh
sudo apt update
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install python3-pip -y
sudo apt install python3.10 python3.10-venv -y
python3.10 -m venv venv
source venv/bin/activate
pip install --upgrade awscli
pip install apache-airflow
pip install apache-airflow-providers-amazon
airflow standalone
```
- **Note down Airflow ID and password**
- **Modify Security Group**:
  - Go to **Instance Security Settings**
  - Add inbound rule: `Custom TCP`, Port `8080`, Source: `Anywhere IPv4`
- **Login to Airflow**

---

## 3. Configure Remote Access via VS Code

### Steps
1. Open **VS Code**
2. Install **Remote-SSH Extension**
3. Connect to EC2 via SSH

---

## 4. Airflow DAG Setup

### Add DAG to Airflow
#### File: `~/airflow/dags/zillowDA.py`
```python
from airflow import DAG
from datetime import datetime, timedelta
import json
import requests
from airflow.operators.python import PythonOperator

with open('/home/ubuntu/airflow/config_api.json', 'r') as config_file:
    api_host_key = json.load(config_file)

now = datetime.now()
dt_now_string = now.strftime("%d%m%Y%H%M%S")

def extract_zillow_data(**kwargs):
    url = kwargs['url']
    headers = kwargs['headers']
    querystring = kwargs['querystring']
    dt_string = kwargs['date_string']
    
    response = requests.get(url, headers=headers, params=querystring)
    response_data = response.json()
    
    output_file_path = f"/home/ubuntu/response_data_{dt_string}.json"
    
    with open(output_file_path, "w") as output_file:
        json.dump(response_data, output_file, indent=4)
    
    return [output_file_path]

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2024, 11, 11),
    'email': ['thatdataguyhet@gmail.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(seconds=15)
}

with DAG('zillow_analytics_dag',
         default_args=default_args,
         schedule_interval='@daily',
         catchup=False) as dag:
    
    extract_zillow_data_var = PythonOperator(
        task_id='tsk_extract_zillow_data',
        python_callable=extract_zillow_data,
        op_kwargs={'url': '', 'querystring': {}, 'headers': api_host_key, 'date_string': dt_now_string}
    )
```

### Create API Config File
#### File: `~/airflow/config_api.json`
- Get API credentials from **RapidAPI** (Zillow API)
- Store them in `config_api.json`

---

## 5. AWS S3 Integration

### Create S3 Bucket
```sh
aws s3 mb s3://zillow-bucket
```

### Update DAG to Upload Data to S3
#### Modify `zillowDA.py`
```python
from airflow.operators.bash import BashOperator

load_to_s3 = BashOperator(
    task_id='tsk_load_to_s3',
    bash_command='aws s3 mv {{ ti.xcom_pull(task_ids="tsk_extract_zillow_data")[0] }} s3://zillow-bucket/',
)

extract_zillow_data_var >> load_to_s3
```

---

## 6. Configure IAM Role for EC2

### Steps
1. **IAM > Create Role**
   - AWS Service: **EC2**
   - Attach Policy: **AmazonS3FullAccess**
   - Role Name: `ec2SeveralAccess-08232023`
2. **Assign Role to EC2 Instance**
   - Select EC2 Instance
   - Actions > Security > Modify IAM Role
   - Select `ec2SeveralAccess-08232023`
   - Update IAM Role

---

## 7. Execute the ETL Pipeline

### Start Airflow & Run DAG
```sh
airflow scheduler &
airflow webserver &
```
- **Trigger DAG in Airflow UI**
- Check **S3 Bucket** for response file

---


---

For any queries, feel free to reach out!

