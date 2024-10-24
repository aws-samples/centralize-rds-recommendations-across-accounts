---
title: centralize amazon rds recommendations
author: hai tran
date: 16/10/2024
---

## Architecture

![arch](./asset/arch.png)

This respository provide a sample code to deploy a solution for collecting and centralizing Amazon RDS recommendations from service owner accounts within an organization. The deployment leverages CloudFormation stack set.

1. The Lambda function in the monitoring account assume role in a service owner account.

2. The Lambda function call Amazon RDS [DescribeDBRecommendations](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DescribeDBRecommendations.html).

3. The Lambda function send recommendations in json format to a S3 bucket.

## Deploy

The key point here is we leverage CloudFormation stack set to deploy an IAM role into each service owner account. Then the Lambda function in the monitoring account will assume this role to get Amazon RDS recommendation in each service owner account.

Step 1. Deploy a Lambda function in the monitoring account.

```bash
aws cloudformation create-stack \
 --stack-name cf-lambda-for-centralize-rds-recommendations \
 --template-body file://cf-lambda.yaml \
 --parameters '{"TargetAccountIds":"<TARGET_ACOCUNT_ID_1>,<TARGET_ACCOUNT_ID_2>"}'
 --capabilities CAPABILITY_NAMED_IAM
```

Step 2. Create a CloudFormation stack-set in management account.

```bash
aws cloudformation create-stack-set \
 --stack-set-name cf-iam-role-for-centralize-rds-recommendations \
 --permission-model SERVICE_MANAGED \
 --auto-deployment '{"Enabled":true,"RetainStacksOnAccountRemoval":false}' \
 --template-body file://cf-iam-role.yaml \
 --parameters '{"ParameterKey": "ServiceRoleArn", "ParameterValue": "arn:aws:iam::<MONITOR_ACCOUNT_ID>:role/LambdaGetRDSRecommendationsRole"}' \
 --capabilities CAPABILITY_NAMED_IAM
```

Step 3. Create the stack set to target accounts within an organization unit.

```bash
aws cloudformation create-stack-instances \
 --stack-set-name cf-iam-role-for-centralize-rds-recommendations \
 --deployment-targets '{"OrganizationalUnitIds":["<OU_ID>"]}' \
 --regions ap-southeast-1 \
 --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=1
```
