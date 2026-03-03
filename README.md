# Send CloudWatch Alarms to Slack with AWS Lambda

Read the full [blog post on cloudonaut.io](https://cloudonaut.io/send-cloudwatch-alarms-to-slack-with-aws-lambda/).

Or use [marbot, a free chatbot ensuring you never miss an alert from Amazon Web Services including CloudWatch](https://marbot.io/).

## Slack setup

1. Start by setting up an incoming webhook integration in your Slack workspace: https://my.slack.com/services/new/incoming-webhook/
2. Select a channel or create a new one
3. Click on *Add Incoming WebHooks integration*
4. You are redirected to a new page where you can see your *Webhook URL*. Copy the value; you will need it soon.

## AWS setup

1. Clone or [download](https://github.com/widdix/cloudwatch-alarm-to-slack/zipball/master/) this repository
2. Create an S3 bucket for deployment artifacts (replace `$UniqueSuffix` with e.g. your username): `aws s3 mb s3://cw-to-slack-$UniqueSuffix --region us-east-1`
3. Install Node.js dependencies: `npm install --omit=dev`
4. Package the Lambda function code (replace `$UniqueSuffix` with e.g. your username): `aws cloudformation package --template-file template.yml --s3-bucket cw-to-slack-$UniqueSuffix --output-template-file packaged.yml --region us-east-1`
5. Deploy the CloudFormation stack (replace `$WebhookURL` with your URL from Slack): `aws cloudformation deploy --template-file packaged.yml --stack-name cloudwatch-alarm-to-slack --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --parameter-overrides "WebhookURL=$WebhookURL" --region us-east-1`

To use a named AWS profile, add `--profile $ProfileName` to each command.

## Testing

You can verify the deployment by invoking the Lambda with a test event:

```sh
aws lambda invoke \
  --function-name $(aws cloudformation describe-stack-resources --stack-name cloudwatch-alarm-to-slack --query 'StackResources[?ResourceType==`AWS::Lambda::Function`].PhysicalResourceId' --output text) \
  --cli-binary-format raw-in-base64-out \
  --payload '{
    "Records": [{
      "EventSource": "aws:sns",
      "Sns": {
        "Subject": "ALARM: Test Alarm",
        "Message": "{\"AlarmName\":\"test-alarm\",\"AlarmDescription\":\"Test\",\"AWSAccountId\":\"123456789012\",\"NewStateValue\":\"ALARM\",\"NewStateReason\":\"This is a test invocation to verify Slack integration.\",\"StateChangeTime\":\"2024-01-01T00:00:00.000+0000\",\"Region\":\"US East (N. Virginia)\",\"OldStateValue\":\"OK\"}"
      }
    }]
  }' \
  /tmp/lambda-response.json
```

A successful invocation (status code 200, no `FunctionError`) means the alert was posted to your Slack channel.
