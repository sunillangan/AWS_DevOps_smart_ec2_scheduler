# AWS_DevOps_smart_ec2_scheduler

Project: Smart EC2 Scheduler & Auto-Stop for Cost Optimization

Goal:

Automatically monitor non-production EC2 instances and stop idle ones based on CPU usage. Alert instance owners via Slack or Email, and stop EC2s automatically if no response is received in time.

⸻

Architecture Components
	•	CloudWatch: To monitor CPU utilization
	•	EventBridge: To trigger automation every few hours
	•	Lambda (Python): Core logic to identify, notify, and stop instances
	•	SNS or Slack: For alerting owners
	•	DynamoDB: To track actions and prevent reprocessing
	•	EC2 Tags: For identifying owner emails or Slack usernames

⸻

Step-by-Step Implementation

⸻

Step 1: Tag Your EC2 Instances

Add the following tags to EC2 instances in non-production:
	•	Environment = dev/test
	•	OwnerEmail = youremail@domain.com
	•	AutoStop = true (Only these will be checked)

⸻

Step 2: Create DynamoDB Table

Table Name: ec2-idle-tracker
Primary Key: InstanceId (String)
Attributes:
	•	LastAlertTime
	•	LastCPUCheck
	•	Status (e.g., AlertSent, Stopped, Skipped)

⸻

Step 3: Create Lambda Function

Function Name: ec2-idle-monitor
Runtime: Python 3.12
Timeout: 3 minutes
Permissions Required:
	•	ec2:DescribeInstances, ec2:StopInstances, ec2:CreateTags
	•	cloudwatch:GetMetricStatistics
	•	dynamodb:PutItem, dynamodb:GetItem
	•	sns:Publish or Slack webhook permissions

⸻

Sample Lambda Code (Slack-based)

import boto3, os, requests
from datetime import datetime, timedelta

ec2 = boto3.client('ec2')
cw = boto3.client('cloudwatch')
ddb = boto3.resource('dynamodb')
table = ddb.Table('ec2-idle-tracker')

SLACK_WEBHOOK = os.environ['SLACK_WEBHOOK_URL']

def lambda_handler(event, context):
    reservations = ec2.describe_instances(Filters=[
        {'Name': 'tag:AutoStop', 'Values': ['true']},
        {'Name': 'instance-state-name', 'Values': ['running']}
    ])['Reservations']
    
    for res in reservations:
        for inst in res['Instances']:
            instance_id = inst['InstanceId']
            tags = {t['Key']: t['Value'] for t in inst.get('Tags', [])}
            email = tags.get('OwnerEmail')
            name = tags.get('Name', instance_id)

            # Get CPU Usage
            metric = cw.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=datetime.utcnow() - timedelta(hours=2),
                EndTime=datetime.utcnow(),
                Period=3600,
                Statistics=['Average']
            )
            datapoints = metric['Datapoints']
            avg_cpu = datapoints[0]['Average'] if datapoints else 0
            
            if avg_cpu < 5:
                # Send Slack Alert
                message = f"*Idle EC2 Alert*\nInstance: `{name}` ({instance_id})\nAvg CPU: {avg_cpu:.2f}%\nNo recent activity. Stopping in 1 hour unless extended."
                payload = {"text": message}
                requests.post(SLACK_WEBHOOK, json=payload)
                
                table.put_item(Item={
                    'InstanceId': instance_id,
                    'LastAlertTime': str(datetime.utcnow()),
                    'Status': 'AlertSent'
                })
            else:
                print(f"{instance_id} is active. Skipping.")

    return {'statusCode': 200, 'body': 'Checked all instances'}

Step 4: Slack Webhook Setup
	1.	Go to Slack > Create a new Incoming Webhook App
	2.	Select channel and get Webhook URL
	3.	Add it to Lambda environment variable: SLACK_WEBHOOK_URL

(Alternative: Use SNS or SES for email instead)

⸻

Step 5: Create EventBridge Rule

Trigger Lambda every 2 or 3 hours using cron syntax:
0 */3 * * ? *   # Every 3 hours
Step 6: (Optional) Auto Stop After Timeout

Create another Lambda triggered by EventBridge every 1 hour to:
	•	Query DynamoDB for Status = AlertSent
	•	If LastAlertTime > 1 hour and no response => ec2.stop_instances

⸻

Step 7: Monitor and Audit

Use CloudWatch logs for all Lambda actions. DynamoDB will act as your audit trail.

What to Write in Resume

Project: Smart EC2 Cost Optimization System using AWS Lambda, CloudWatch, Slack
	•	Built a real-time alerting & automation pipeline to stop idle EC2s
	•	Used CloudWatch Metrics + Lambda to identify low CPU usage
	•	Integrated Slack to notify instance owners
	•	Logged all actions in DynamoDB for audit tracking
	•	Reduced dev environment EC2 costs by ~30% monthly
