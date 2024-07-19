
# 🌩️ Cloud Cost Optimizer 🌩️

## Project Description 📜
This project aims to optimize cloud costs by identifying and deleting stale resources such as unused EBS snapshots and forgotten S3 buckets. We'll use AWS Lambda, and CloudWatch to automate this process.

## Table of Contents 📑
- [Introduction](#introduction)
- [Examples](#examples)
- [Solution Overview](#solution-overview)
- [Project Setup](#project-setup)
  - [Step 1: Create EC2 Instance and Snapshot](#step-1-create-ec2-instance-and-snapshot)
  - [Step 2: Create Lambda Function](#step-2-create-lambda-function)
  - [Step 3: Configure Lambda Permissions](#step-3-configure-lambda-permissions)
  - [Step 4: Test the Lambda Function](#step-4-test-the-lambda-function)
  - [Step 5: Automate with CloudWatch](#step-5-automate-with-cloudwatch)
- [Conclusion](#conclusion)

## Introduction 🎯
Cloud cost optimization is crucial to avoid unnecessary expenses. This project automates the identification and deletion of unused resources to reduce costs efficiently.

## Examples 🔍
1. **EC2 Instance and Snapshot**:
   - Creating an EC2 instance automatically creates a volume. If a snapshot of this volume is created and the EC2 instance is later deleted, the snapshot remains, incurring costs.
   
2. **Forgotten S3 Buckets**:
   - An S3 bucket created for temporary use might be forgotten, leading to ongoing storage costs.

## Solution Overview 🛠️
1. **Notifications**: Use AWS SNS to notify users about unused resources.
2. **Automated Deletion**: Use AWS Lambda to automatically delete stale resources.

In this project, we use 2 approaches to manage cloud costs:

## Project Setup 🏗️

### Step 1: Create EC2 Instance and Snapshot 💻
#### Create EC2 Instance
1. Sign in to the [AWS Management Console](https://aws.amazon.com/console/).
2. Navigate to the **EC2 Dashboard**.
3. Click **Launch Instance**.
4. Select an **Amazon Machine Image (AMI)** (e.g., Amazon Linux 2).
5. Choose an **Instance Type** (e.g., t2.micro).
6. Configure the **Instance Details** (default settings are usually sufficient).
7. Add **Storage** (default settings create a root volume).
8. Add **Tags** (optional).
9. Configure **Security Group**:
   - Add rules to allow SSH access (port 22).
10. Review and **Launch** the instance.
11. Select or create a **key pair** for SSH access and click **Launch Instances**.

#### Create Snapshot
1. Navigate to the **EC2 Dashboard**.
2. In the left-hand menu, click on **Volumes** under **Elastic Block Store**.
3. Select the volume attached to your EC2 instance.
4. Click **Actions** and choose **Create Snapshot**.
5. Provide a **Description** (optional) and click **Create Snapshot**.

### Step 2: Create Lambda Function 📝
1. Go to the [AWS Lambda Console](https://console.aws.amazon.com/lambda/).
2. Click **Create function**.
3. Choose **Author from scratch**.
4. Enter a **Function name** and choose a **Runtime** (e.g., Python 3.8).
5. Click **Create function**.
6. In the function code editor, replace the default code with the following:

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

7. Click **Deploy** to save the function.

### Step 3: Configure Lambda Permissions 🔒
1. Go to the Lambda function's **Configuration** tab.
2. Under **Execution role**, click on the role name to open the IAM console.
3. In the IAM role, click **Add permissions** and choose **Attach policies**.
4. Search for and select the **AmazonEC2FullAccess** policy.
5. Click **Attach policy**.

### Step 4: Test the Lambda Function 🧪
1. Go back to the Lambda function.
2. Click **Test** and configure a new test event (you can use the default settings).
3. Click **Create** and then **Test** to execute the function.
4. If the function fails due to a timeout, go to the **Configuration** tab and increase the timeout to 10 seconds.
5. Ensure the function has the necessary permissions to delete snapshots and describe volumes.

### Step 5: Automate with CloudWatch ⏰
1. Go to the [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/).
2. In the left-hand menu, click on **Rules** under **Events**.
3. Click **Create rule**.
4. In the **Event Source** section, choose **Event Source** as **Schedule** and set the frequency (e.g., daily).
5. In the **Targets** section, click **Add target** and select **Lambda function**.
6. Choose the Lambda function you created.
7. Click **Configure details**, provide a name and description, and click **Create rule**.

## Conclusion 🎉
By following these steps, you can efficiently manage cloud costs by identifying and deleting stale resources using AWS Lambda, and CloudWatch. This automated approach ensures you only pay for the resources you actively use.



