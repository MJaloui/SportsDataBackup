![github-header-image (1)](https://github.com/user-attachments/assets/fb42157b-093a-420d-8b69-372b0c74496f)

&nbsp;

&nbsp;

&nbsp;

&nbsp;

# SportsDataBackup
BackupHighlights automates fetching sports highlights, stores data in S3 and DynamoDB, processes videos, and runs on a schedule using ECS Fargate and EventBridge. It uses templated JSON files with environment variable injection for easy configuration and deployment.

# Prerequisites
Before running the scripts, ensure you have the following:

## **1** Create Rapidapi Account
Rapidapi.com account, will be needed to access highlight images and videos.

For this example we will be using NCAA (USA College Basketball) highlights since it's included for free in the basic plan.

[Sports Highlights API](https://rapidapi.com/highlightly-api-highlightly-api-default/api/sport-highlights-api/playground/apiendpoint_16dd5813-39c6-43f0-aebe-11f891fe5149) is the endpoint we will be using 

## **2** Verify prerequites are installed 

Docker should be pre-installed in most regions 

```bash
docker --version
```

![image](https://github.com/user-attachments/assets/5c5afb20-99a3-4b26-a2d1-b08c2a9ca382)


AWS CloudShell has AWS CLI pre-installed 

```bash
aws --version
```

![image](https://github.com/user-attachments/assets/1b919ca9-616a-4df5-beb5-19ba5549ddea)


Python3 should be pre-installed also

```bash
python3 --version
```

![image](https://github.com/user-attachments/assets/499dc09f-126c-4485-843d-4a6309c9f68b)

Install gettext package - envsubst is a command-line utility is used for environment variable substituition in shell scripts and text files.

[Install Steps](https://www.drupal.org/docs/8/modules/potion/how-to-install-setup-gettext) 

    - Enter the command below, the "y" to confirm information it's ok.

```bash
sudo yum install gettext"
```

![image](https://github.com/user-attachments/assets/1374dd09-ddee-46f7-947c-f8a22d9adb07)

![image](https://github.com/user-attachments/assets/607288f4-ce16-446b-b7cd-e9c135c53659)

## **3** Retrieve AWS Account ID

Copy your AWS Account ID Once logged in to the AWS Management Console Click on your account name in the top right corner You will see your account ID Copy and save this somewhere safe because you will need to update codes in the labs later

## **4** Retrieve Access Keys and Secret Access Keys
You can check to see if you have an access key in the IAM dashboard
Under Users, click on a user and then "Security Credentials"
Scroll down until you see the Access Key section
You will not be able to retrieve your secret access key so if you don't have that somewhere, you need to create an access key.


# START HERE 
## **Step 1: Clone The Repo**
```bash
git clone https://github.com/MJaloui/SportsDataBackup
cd src
```

![image](https://github.com/user-attachments/assets/a52abc33-f62f-48c6-8a4d-ec8b995aa9f5)


## **Step 2: Create and Configure the .env File**
Search, then Replace the following values in the .env file:

1. Your-AWS-Account-ID
  - Use the command below to find AWS ID or click on your username in the top right to view ID.

```bash
aws sts get-caller-identity --query "Account" --output text
```

2. Your-RAPIDAPI-Key
3. Your-AWS-Access-Key
4. Your-AWS-Secret-Access-key
5. S3_BUCKET_NAME=your-alias
6. Your-MediaConvert-Endpoint

  - Enter the command below to find mediaconvert endpoint.

```bash
aws mediaconvert describe-endpoints
```

7. SUBNET_ID=subnet-<Your-SubnetId> 

8. SECURITY_GROUP_ID=sg-<Your-SecurityGroupId>

Steps for SubnetID and Security Group ID:

1. In the github repo, there is a resources folder with "vpc_setup.sh" file inside, copy the entire contents.

2. In the AWS Cloudshell or vs code terminal, create the file vpc_setup.sh.

  - Paste the script inside, save the file, and exit.

![image](https://github.com/user-attachments/assets/a0385afc-b300-42ab-9d65-8f3a310c6c88)

![image](https://github.com/user-attachments/assets/b4cf4cbb-2bc0-4b8a-b2ac-50548b89b768)

![image](https://github.com/user-attachments/assets/5f4644d5-1acc-4930-a8fe-c4531f43f1c6)

3. Run the script

```bash
bash vpc_setup.sh
```
![image](https://github.com/user-attachments/assets/c246acf4-06e3-460f-9435-56810c7d3431)


4. You will see variables in the output, paste these variables into Subnet_ID and Security_Group_ID in the ".env" file.


## **Step 3: Load Environment Variables**
```bash
set -a
source .env
set +a
```

![image](https://github.com/user-attachments/assets/85eb3d85-bf73-4953-8187-af1f7b270bb7)


Optional - Verify the variables are loaded

```bash
echo $AWS_LOGS_GROUP
echo $TASK_FAMILY
echo $AWS_ACCOUNT_ID
```

![image](https://github.com/user-attachments/assets/3adedb3f-9b21-4fc7-a717-eb5a2249ee80)



## **Step 4: Generate Final JSON Files from Templates. If "gettext" is not installed, you will recieve an error**

1. ECS Task Definition

```bash
envsubst < taskdef.template.json > taskdef.json
```

![image](https://github.com/user-attachments/assets/8bcfb91c-c9e1-4b68-b3f4-f69ad7d24ba0)


2. S3/DynamoDB Policy

```bash
envsubst < s3_dynamodb_policy.template.json > s3_dynamodb_policy.json
```

![image](https://github.com/user-attachments/assets/4f2d83e3-cadf-446d-aba4-73f88ba2fc38)


3. ECS Target

```bash
envsubst < ecsTarget.template.json > ecsTarget.json
```

![image](https://github.com/user-attachments/assets/67382bc8-31bb-4488-b40d-6daf79f83716)


4. ECS Events Role Policy
```bash
envsubst < ecseventsrole-policy.template.json > ecseventsrole-policy.json
```

![image](https://github.com/user-attachments/assets/2360fcba-84ed-4097-84ee-a5ca5d323acc)



*Optional - Open the generated files using cat or a text editor to confirm that all place holders have been correctly replaced

## **Step 5: Build and Push Docker Image**

1. Create an ECR Repo

```bash
aws ecr create-repository --repository-name sports-backup
```

![image](https://github.com/user-attachments/assets/99f813ca-df58-4e47-a3a8-a7f8c6ffcabd)


2.Log In To ECR

```bash
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

![image](https://github.com/user-attachments/assets/2eb156ca-aeb8-49c1-954f-941a05d0aa9f)

3. Build the Docker Image

```bash
docker build -t sports-backup .
```

![image](https://github.com/user-attachments/assets/0a47e928-75e6-40aa-a667-abb5109134a7)


4.Tag the Image for ECR

```bash
docker tag sports-backup:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
```

![image](https://github.com/user-attachments/assets/86c6df68-7812-409f-8065-f4a8b1e8703f)


5. Push the Image

```bash
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
```

![image](https://github.com/user-attachments/assets/9a8393b1-39d8-436f-97fd-84202cbe3f68)


## **Step 6: Create AWS Resources**

1. Register the ECS Task Definition

```bash
aws ecs register-task-definition --cli-input-json file://taskdef.json --region ${AWS_REGION}
```

![image](https://github.com/user-attachments/assets/be37f540-0efa-4670-aefc-9ba9e23d0407)


2. Create the CloudWatch Logs Group

```bash
aws logs create-log-group --log-group-name "${AWS_LOGS_GROUP}" --region ${AWS_REGION}
```

![image](https://github.com/user-attachments/assets/8357ebe9-004d-4c52-8564-12965a787cab)


3. Attach the S3/DynamoDB Policy to the ECS Task Execution Role

```bash
aws iam put-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name S3DynamoDBAccessPolicy \
  --policy-document file://s3_dynamodb_policy.json
```

![image](https://github.com/user-attachments/assets/16b525e6-c555-4719-bed0-5a8222965283)


4. Set up the ECS Events Role
Create the Role with Trust Policy

```bash
aws iam create-role --role-name ecsEventsRole --assume-role-policy-document file://ecsEventsRole-trust.json
```

![image](https://github.com/user-attachments/assets/cd9b0c0c-c2ba-46e2-ad7b-558b9b402927)


5. Attach the Events Role Policy

```bash
aws iam put-role-policy --role-name ecsEventsRole --policy-name ecsEventsPolicy --policy-document file://ecseventsrole-policy.json
```

![image](https://github.com/user-attachments/assets/a98bb367-58a8-4174-9fc5-8bac4e1a48b5)



## **Step 7: Create an EventBridge Rule to Schedule the Task**
1. Create the Rule
```bash
aws events put-rule --name SportsBackupScheduleRule --schedule-expression "rate(1 day)" --region ${AWS_REGION}
```

![image](https://github.com/user-attachments/assets/a3ab2e60-569b-4c9d-9299-6142285b9c85)


2. Add the Target

```bash
aws events put-targets --rule SportsBackupScheduleRule --targets file://ecsTarget.json --region ${AWS_REGION}
```

![image](https://github.com/user-attachments/assets/65feebbe-cf03-4416-ad4b-cd2bd8fa7256)



## **Step 8. Create cluster**

```bash
aws ecs create-cluster --cluster-name sports-backup-cluster --region ${AWS_REGION}
```

![image](https://github.com/user-attachments/assets/2c93f69b-0f93-4285-be5a-1ae760807f22)


## **Step 9: Manually Test ECS Task**

```bash
aws ecs run-task \
  --cluster sports-backup-cluster \
  --launch-type Fargate \
  --task-definition ${TASK_FAMILY} \
  --network-configuration "awsvpcConfiguration={subnets=[\"${SUBNET_ID}\"],securityGroups=[\"${SECURITY_GROUP_ID}\"],assignPublicIp=\"ENABLED\"}" \
  --region ${AWS_REGION}
```

![image](https://github.com/user-attachments/assets/13c270c0-ab84-4f73-a00c-8758537f06c0)


## **Step 10. Verify the cluster*

    - Go to Elastic Container Services > Clusters > Sports-backup-cluster > Task

![image](https://github.com/user-attachments/assets/9a339f37-c874-4e81-9c5a-27604fd938e4)

![image](https://github.com/user-attachments/assets/ff97703d-3e5c-4090-95f4-08b81aee9130)
  
![image](https://github.com/user-attachments/assets/1fb4182d-aaf6-4f8c-9471-c9b1c99117ff)


### **What We Learned**
1. Using templates to generate json files
2. Integrating DynamoDB to store data backup
3. Cloudwatcher for logging

### **Future Enhancements**
1. Integrate exporting a table from DynamoDB to an S3 bucket
2. Configure an automated backup
3. Creating batch processing of the entire Json file (importing more than 10 videos)
