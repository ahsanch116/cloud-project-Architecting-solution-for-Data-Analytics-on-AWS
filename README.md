# Architecting Solutions: Project for Data Analytics on AWS

This README provides a detailed guide on building a data analytics solution using AWS managed services. The solution is designed to ingest, store, and visualize clickstream data, providing insights into the restaurant's menu items. The guide walks through the setup process, from creating IAM roles to visualizing data with Amazon QuickSight.

## Table of Contents
- [Introduction](#introduction)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Task 1: Creating IAM Policy and Role](#task-1-creating-iam-policy-and-role)
  - [Task 2: Creating an S3 Bucket](#task-2-creating-an-s3-bucket)
  - [Task 3: Creating a Lambda Function](#task-3-creating-a-lambda-function)
  - [Task 4: Creating a Kinesis Data Firehose Delivery Stream](#task-4-creating-a-kinesis-data-firehose-delivery-stream)
  - [Task 5: Adding Firehose Delivery Stream ARN to the S3 Bucket](#task-5-adding-firehose-delivery-stream-arn-to-the-s3-bucket)
  - [Task 6: Creating an API in API Gateway](#task-6-creating-an-api-in-api-gateway)
  - [Task 7: Creating an Athena Table](#task-7-creating-an-athena-table)
  - [Task 8: Visualizing Data with QuickSight](#task-8-visualizing-data-with-quicksight)
  - [Task 9: Deleting Resources](#task-9-deleting-resources)
- [Conclusion](#conclusion)
- [License](#license)

## Introduction

This proof of concept (PoC) guides you through creating a data analytics solution using AWS services. The solution ingests clickstream data, processes it, stores it in S3, and visualizes it using QuickSight, providing valuable insights into customer interactions with menu items.

## Architecture Overview

The architecture for this solution includes the following components:

1. **API Gateway**: Ingests clickstream data.
2. **AWS Lambda**: Transforms the incoming data.
3. **Kinesis Data Firehose**: Streams data to an S3 bucket.
4. **Amazon S3**: Stores the clickstream data.
5. **Amazon Athena**: Queries the data stored in S3.
6. **Amazon QuickSight**: Visualizes the data for analysis.

## [Architecture Diagram]
![Network Diagrams](https://github.com/user-attachments/assets/3ccfb2f0-4e0f-4003-991a-19fabd17db54)


## Prerequisites

- An active AWS account with access to the AWS Management Console.
- Familiarity with AWS services such as IAM, Lambda, S3, Kinesis, API Gateway, Athena, and QuickSight.
- Permissions to create and manage resources in AWS.

## Setup Instructions

### Task 1: Creating IAM Policy and Role

#### Step 1.1: Creating Custom IAM Policies

1. Sign in to the AWS Management Console.
2. Navigate to the IAM service.
3. Create a new policy with the following JSON:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "VisualEditor0",
               "Effect": "Allow",
               "Action": "firehose:PutRecord",
               "Resource": "*"
           }
       ]
   }
   ```
4. Name the policy `API-Firehose` and create it.

#### Step 1.2: Creating an IAM Role

1. Navigate to IAM Roles in the IAM dashboard.
2. Create a role with the following configuration:
   - **Trusted entity**: AWS service.
   - **Use case**: API Gateway.
3. Name the role `APIGateway-Firehose` and attach the `API-Firehose` policy.

### Task 2: Creating an S3 Bucket

1. In the AWS Management Console, open the S3 service.
2. Create a new bucket with a unique name, e.g., `architecting-week2-<your initials>`.
3. Note the S3 bucket's ARN for later use.

### Task 3: Creating a Lambda Function

1. In the Lambda service, create a function using the `Kinesis` blueprint with Python 3.8.
2. Replace the function code with:
   ```python
   import json
   import boto3
   import base64

   output = []

   def lambda_handler(event, context):
       for record in event['records']:
           payload = base64.b64decode(record['data']).decode('utf-8')
           row_w_newline = payload + "\n"
           row_w_newline = base64.b64encode(row_w_newline.encode('utf-8'))
           output_record = {
               'recordId': record['recordId'],
               'result': 'Ok',
               'data': row_w_newline
           }
           output.append(output_record)
       return {'records': output}
   ```
3. Deploy the function and set the timeout to 10 seconds.

### Task 4: Creating a Kinesis Data Firehose Delivery Stream

1. Create a Kinesis Data Firehose delivery stream with the following configuration:
   - **Source**: Direct PUT
   - **Destination**: Amazon S3
   - **Data Transformation**: Enabled with the Lambda function created earlier.

2. Save the delivery stream's IAM role ARN for use in subsequent steps.

### Task 5: Adding Firehose Delivery Stream ARN to the S3 Bucket

1. Edit the S3 bucket policy to allow access from the Kinesis Data Firehose by adding the following policy:
   ```json
   {
       "Version": "2012-10-17",
       "Id": "PolicyID",
       "Statement": [
           {
               "Sid": "StmtID",
               "Effect": "Allow",
               "Principal": {
                   "AWS": "arn:aws:iam::<account ID>:role/service-role/KinesisFirehoseServiceRole-PUT-S3-..."
               },
               "Action": [
                   "s3:AbortMultipartUpload",
                   "s3:GetBucketLocation",
                   "s3:GetObject",
                   "s3:ListBucket",
                   "s3:ListBucketMultipartUploads",
                   "s3:PutObject",
                   "s3:PutObjectAcl"
               ],
               "Resource": [
                   "arn:aws:s3:::DOC-EXAMPLE-BUCKET",
                   "arn:aws:s3:::DOC-EXAMPLE-BUCKET/*"
               ]
           }
       ]
   }
   ```

### Task 6: Creating an API in API Gateway

1. Create a REST API named `clickstream-ingest-poc`.
2. Set up a POST method under a resource named `poc` with the following integration:
   - **Integration type**: AWS Service
   - **AWS Service**: Firehose
   - **Action**: PutRecord
   - **Execution role**: Use the ARN of the `APIGateway-Firehose` role.
3. Configure the method to pass through JSON data to the Firehose delivery stream.

### Task 7: Creating an Athena Table

1. In the Athena service, set up a query result location pointing to your S3 bucket.
2. Create a table using the following SQL:
   ```sql
   CREATE EXTERNAL TABLE my_ingested_data (
       element_clicked STRING,
       time_spent INT,
       source_menu STRING,
       created_at STRING
   )
   PARTITIONED BY (
       datehour STRING
   )
   ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
   with serdeproperties ( 'paths'='element_clicked, time_spent, source_menu, created_at' )
   LOCATION "s3://<Your S3 Bucket Name>/"
   TBLPROPERTIES (
       "projection.enabled" = "true",
       "projection.datehour.type" = "date",
       "projection.datehour.format" = "yyyy/MM/dd/HH",
       "projection.datehour.range" = "2021/01/01/00,NOW",
       "projection.datehour.interval" = "1",
       "projection.datehour.interval.unit" = "HOURS",
       "storage.location.template" = "s3://<Your S3 Bucket Name>/${datehour}/"
   )
   ```
3. Run a query to ensure that data is being ingested correctly.

### Task 8: Visualizing Data with QuickSight

1. Set up QuickSight to access your S3 bucket and Athena workgroup.
2. Create a new dataset in QuickSight using Athena as the data source.
3. Visualize the data by creating charts and graphs.

### Task 9: Deleting Resources

To avoid incurring costs, delete all resources created during this exercise:
- Delete QuickSight dashboards and the QuickSight account if not needed.
- Delete the S3 bucket, Athena table, API Gateway, Kinesis Data Firehose stream, Lambda function, and IAM roles/policies.

## Conclusion

This guide provided step-by-step instructions for setting up a data analytics solution using AWS services. By following these steps, you can gain insights into customer interactions with your restaurant's menu, helping you make data-driven decisions.
