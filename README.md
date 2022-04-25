### Get daily AWS cost usage alerts on Slack using Lambda.

source: https://iamops.io/get-aws-pricing-alerts-on-slack-2/

To include requests python library:
```
pip install requests -t ./
chmod -R 755 .
zip -r ../request_library.zip .
```

Create Lambda Python and upload the generated request_library.zip file.

Add a file to lambda lambda_function.py with below content.

Change account ID, Slack webhook and if needed start date, end date and COST filters. Currently this displays charge from the 1st of the month to today.

```
import json
import boto3
import requests
from datetime import datetime, timedelta


CLIENT = boto3.client('ce')

SLACK_WEBHOOK_URL = 'https://hooks.slack.com/services/...' # Enter the slack webhook url.

def lambda_handler(event, context):
    START_DATE = datetime.today().replace(day=1).strftime('%Y-%m-%d')
    END_DATE = datetime.today().strftime('%Y-%m-%d')
    AWS_ACCOUNT_ID = ['xxxxxxxxx'] # Enter your aws account id. If you have multiple accounts put all account ids.
    
    for account_id in AWS_ACCOUNT_ID:
        COST = CLIENT.get_cost_and_usage(
            TimePeriod = {'Start': START_DATE, 'End': END_DATE},
            Granularity = 'MONTHLY',
            Metrics = ['UnblendedCost'],
            Filter = {'Dimensions': {
                'Key': 'LINKED_ACCOUNT', 'Values': [account_id]
            }}
        )
        
        TOTAL_COST = round(float(COST['ResultsByTime'][0]['Total']['UnblendedCost']['Amount']), 2)
        UNIT = COST['ResultsByTime'][0]['Total']['UnblendedCost']['Unit']
        
        SLACK_MESSAGE = {
            "channel": "#st_aws-feed",
            "username": "AWS Cost Monitor",
            "text": f"Your AWS account {account_id} current monthly cost usage is {TOTAL_COST} {UNIT}.",
            "icon_emoji": ":ghost:",
        }
        ENCODED_SLACK_MESSAGE = json.dumps(SLACK_MESSAGE).encode('utf-8')
        
        SLACK_RESPONSE = requests.post(SLACK_WEBHOOK_URL, ENCODED_SLACK_MESSAGE)
        print(SLACK_RESPONSE.text)
```

Create IAM policy with below code:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ce:GetCostAndUsageWithResources",
                "ce:GetCostAndUsage"
            ],
            "Resource": "*"
        }
    ]
}
```

Attach above policy to the role of the created lambda.

If you want this lambda to run daily create an event Bridge and point to this lambda.
