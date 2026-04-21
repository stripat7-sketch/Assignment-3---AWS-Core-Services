# ITCS-6190 Assignment 3: AWS Data Processing Pipeline

This project demonstrates an end-to-end serverless data processing pipeline on AWS. The process involves ingesting raw data into S3, using a Lambda function to process it, cataloging the data with AWS Glue, and finally, querying and visualizing the results on a dynamic webpage hosted on an EC2 instance.

## 1. Amazon S3 Bucket Structure 🪣

First, set up an S3 bucket with the following folder structure to manage the data workflow:

* **`pipe-ama-s3-stripat/`**
    * **`raw/`**: For incoming raw data files.
    * **`processed/`**: For cleaned and filtered data output by the Lambda function.
    * **`enriched/`**: For storing athena query results.

<img width="1879" height="939" alt="Amazon S3 Bucket Structure" src="https://github.com/user-attachments/assets/342e38b7-a9fa-49db-b61e-7c8591802315" />

---

## 2. IAM Roles and Permissions 🔐

Create the following IAM roles to grant AWS services the necessary permissions to interact with each other securely.

### 2.1 Lambda Execution Role
- **Role Name:** `Lambda-S3-Processing-Role`
- **Trusted Entity:** Lambda
- **Attached Policies:**
  - `AWSLambdaBasicExecutionRole`
  - `AmazonS3FullAccess`

### 2.2 Glue Service Role
- **Role Name:** `Glue-S3-Crawler-Role`
- **Trusted Entity:** Glue
- **Attached Policies:**
  - `AmazonS3FullAccess`
  - `AWSGlueConsoleFullAccess`
  - `AWSGlueServiceRole`

### 2.3 EC2 Instance Profile
- **Role Name:** `EC2-Athena-Dashboard-Role`
- **Trusted Entity:** EC2
- **Attached Policies:**
  - `AmazonS3FullAccess`
  - `AmazonAthenaFullAccess`
 
### Approach:
- Used least privilege principle by attaching only necessary policies
- Each role is specific to its service use case
- Created roles before creating the services to ensure smooth setup

<img width="1881" height="941" alt=" IAM roles " src="https://github.com/user-attachments/assets/a1f2c61d-696e-4888-ad9a-4c50960895ad" />

---

## Step 3: Lambda Function ⚙️

I created a Lambda function to automatically process files uploaded to the `raw/` S3 folder.

### Function Details:
- **Function Name:** `FilterAndProcessOrders`
- **Runtime:** Python 3.9
- **Execution Role:** `Lambda-S3-Processing-Role`

### Processing Logic:
The Lambda function:
1. Reads the raw CSV file from S3
2. Filters out orders that are:
   - Status = "pending" OR "cancelled"
   - AND OrderDate older than 30 days
3. Saves the filtered results to the `processed/` folder

### Key Code Snippet:
```python
def lambda_handler(event, context):
    # Get S3 bucket and object key from event
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    raw_key = event['Records'][0]['s3']['object']['key']
    
    # Filter records
    for row in reader:
        if order_status not in ['pending', 'cancelled'] or order_date > cutoff_date:
            filtered_rows.append(row)
    
    # Save to processed folder
    processed_key = f"processed/filtered_{file_name}"
    s3.put_object(Bucket=bucket_name, Key=processed_key, Body=output.getvalue())
```
<img width="1880" height="939" alt="Lambda function" src="https://github.com/user-attachments/assets/f800fb34-a7f6-46c8-bf84-e867a37e2925" />
---

## 4. Configure the S3 Trigger ⚡

Set up the S3 trigger to invoke your Lambda function automatically.

1.  In the Lambda function overview, click **+ Add trigger**.
2.  **Source**: Choose **S3**.
3.  **Bucket**: Select your S3 bucket.
4.  **Event types**: Choose **All object create events**.
5.  **Prefix (Required)**: Enter `raw/`. This ensures the function only triggers for files in this folder.
6.  **Suffix (Recommended)**: Enter `.csv`.
7.  Check the acknowledgment box and click **Add**.

<img width="1882" height="939" alt="Trigger Configured" src="https://github.com/user-attachments/assets/74664d95-1d16-4d28-b9c4-851fd504a575" />

--- 
**Start Processing of Raw Data**: Now upload the Orders.csv file into the `raw/` folder of the S3 Bucket. This will automatically trigger the Lambda function.
---

<img width="1881" height="941" alt="Processed CSV File" src="https://github.com/user-attachments/assets/acdda4b7-e193-42a7-bcd0-cb445a616ca4" />

## 5. Create a Glue Crawler 🕸️

I created a Glue Crawler to scan the processed data and create a data catalog, making it queryable by Athena.

### Crawler Configuration:
- **Name:** `orders_processed_crawler`
- **Data Source:** S3 (`s3://pipe-ama-s3-stripat/processed/`)
- **IAM Role:** `Glue-S3-Crawler-Role`
- **Output Database:** `orders_db`
- **Schedule:** On demand

### Table Created:
- **Table Name:** `filtered_orders_csv`
- **Columns:** OrderID, Customer, Amount, Status, OrderDate

### CloudWatch Logs:
The crawler successfully identified the CSV schema and created the table.
<img width="1915" height="942" alt="Crawler CloudWatch" src="https://github.com/user-attachments/assets/254f6162-56ff-43f6-b4a9-1a87c9e11718" />

---

## 6. Amazon Athena Queries 🔍

I used Athena to run SQL queries on the processed data.

### Setup:
- **Database:** `orders_db`
- **Table:** `filtered_orders_csv`
- **Query Result Location:** `s3://pipe-ama-s3-stripat/enriched/`

### Queries Executed:

#### Query 1: Total Sales by Customer
```sql
SELECT Customer, SUM(Amount) AS TotalAmountSpent
FROM filtered_orders_csv
GROUP BY Customer
ORDER BY TotalAmountSpent DESC;
```
#### Query 2: Monthly Order Volume and Revenue
```sql
SELECT DATE_TRUNC('month', CAST(OrderDate AS DATE)) AS OrderMonth,
       COUNT(OrderID) AS NumberOfOrders,
       ROUND(SUM(Amount), 2) AS MonthlyRevenue
FROM filtered_orders_csv
GROUP BY 1 ORDER BY OrderMonth;
```
#### Query 3: Order Status Dashboard
```sql
SELECT Status, COUNT(OrderID) AS OrderCount, ROUND(SUM(Amount), 2) AS TotalAmount
FROM filtered_orders_csv
GROUP BY Status;
```
#### Query 4: Average Order Value per Customer
```sql
SELECT Customer, ROUND(AVG(Amount), 2) AS AverageOrderValue
FROM filtered_orders_csv
GROUP BY Customer
ORDER BY AverageOrderValue DESC;
```
#### Query 5: Top 10 Largest Orders in February 2025
```sql
SELECT OrderDate, OrderID, Customer, Amount
FROM filtered_orders_csv
WHERE CAST(OrderDate AS DATE) BETWEEN DATE '2025-02-01' AND DATE '2025-02-28'
ORDER BY Amount DESC LIMIT 10;
```
<img width="1879" height="939" alt="All 5 Athena Queries" src="https://github.com/user-attachments/assets/91cf2691-df50-4c07-a389-2bcefbfa72d4" />

---

## 7. Launch the EC2 Web Server 🖥️

This instance will host a simple web page to display the Athena query results.

1.  Navigate to the **EC2** service and click **Launch instance**.
2.  **Name**: `Athena-Dashboard-Server`.
3.  **Application and OS Images**: Select **Amazon Linux 2023 AMI**.
4.  **Instance type**: Choose **t2.micro** (Free tier eligible).
5.  **Key pair (login)**: Create and download a new key pair. **Save the `.pem` file!**
6.  **Network settings**: Click **Edit** and configure the security group:
    * **Rule 1 (SSH)**: Type: `SSH`, Port: `22`, Source: `My IP`.
    * **Rule 2 (Web App)**: Click **Add security group rule**.
        * Type: `Custom TCP`
        * Port Range: `5000`
        * Source: `Anywhere` (`0.0.0.0/0`)
7.  **Advanced details**: Scroll down and for **IAM instance profile**, select the **EC2 Instance Profile** you created.
8.  Click **Launch instance**.

---

## 8. Connect to Your EC2 Instance

1.  From the EC2 dashboard, select your instance and copy its **Public IPv4 address**.
2.  Open a terminal or SSH client and connect using your key pair:

    ```bash
    ssh -i /path/to/your-key-file.pem ec2-user@YOUR_PUBLIC_IP_ADDRESS
    ```

---

## 9. Set Up the Web Environment

Once connected via SSH, run the following commands to install the necessary software.

1.  **Update system packages**:
    ```bash
    sudo yum update -y
    ```
2.  **Install Python and Pip**:
    ```bash
    sudo yum install python3-pip -y
    ```
3.  **Install Python libraries (Flask & Boto3)**:
    ```bash
    pip3 install Flask boto3
    ```

---

## 10. Create and Configure the Web Application

1.  Create the application file using the `nano` text editor:
    ```bash
    nano app.py
    ```
2.  Copy and paste your Python web application code (`EC2InstanceNANOapp.py`) into the editor.

3.  ‼️ **Important**: Update the placeholder variables at the top of the script:
    * `AWS_REGION`: Your AWS region (e.g., `us-east-1`).
    * `ATHENA_DATABASE`: The name of your Glue database (e.g., `orders_db`).
    * `S3_OUTPUT_LOCATION`: The S3 URI for your Athena query results (e.g., `s3://your-athena-results-bucket/`).

4.  Save the file and exit `nano` by pressing `Ctrl + X`, then `Y`, then `Enter`.

---

## 11. Run the App and View Your Dashboard! 🚀

1.  Execute the Python script to start the web server:
    ```bash
    python3 app.py
    ```
    You should see a message like `* Running on http://0.0.0.0:5000/`.

2.  Open a web browser and navigate to your instance's public IP address on port 5000:
    ```
    http://YOUR_PUBLIC_IP_ADDRESS:5000
    ```
    You should now see your Athena Orders Dashboard!

<img width="1471" height="843" alt="Final Dashboard Webpage" src="https://github.com/user-attachments/assets/97bd7def-b062-48f9-af52-ea7b886e967f" />


---

## Important Final Notes

* **Stopping the Server**: To stop the Flask application, return to your SSH terminal and press `Ctrl + C`.
* **Cost Management**: This setup uses free-tier services. To prevent unexpected charges, **stop or terminate your EC2 instance** from the AWS console when you are finished.

---
## Challenges Encountered and Solutions

| Challenge | Solution |
|-----------|----------|
| Lambda function not triggering | Added proper prefix `raw/` and suffix `.csv` filters |
| Athena table not found | Table was named `filtered_orders_csv` instead of `filtered_orders` - updated queries accordingly |
| EC2 region mismatch | EC2 was in Ohio (us-east-2) but S3/Athena were in N. Virginia (us-east-1) - updated app.py region to us-east-1 |
| S3 output location error | Created `enriched/` folder and ensured proper IAM permissions |
| SSH permission denied | Fixed key file permissions using `chmod 400` |

<img width="1474" height="874" alt="Dashboard Error(due to region change)" src="https://github.com/user-attachments/assets/80835004-bd18-4abf-9657-4bc7cea775e0" />


---
## Cost Management

To avoid unnecessary AWS charges, I will:

1. Terminate EC2 instance
2. Delete S3 bucket contents
3. Delete Lambda function
4. Delete Glue crawler and database
5. Delete IAM roles
6. Delete key pair

---

## Conclusion

This assignment successfully demonstrated an end-to-end serverless data processing pipeline on AWS. Key takeaways:

- **S3** provides scalable object storage
- **Lambda** enables serverless compute for event-driven processing
- **Glue** automates data cataloging
- **Athena** allows serverless SQL queries on S3 data
- **EC2** hosts web applications to visualize results

The pipeline is fully automated, cost-effective, and follows AWS best practices for security and scalability.

---

## References

- AWS Lambda Documentation
- Amazon S3 Documentation
- AWS Glue Documentation
- Amazon Athena Documentation
- Amazon EC2 Documentation
---
