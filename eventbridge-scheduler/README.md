# EventBridge Scheduler POC

EventBridge Scheduler is a separate service from EventBridge Rules. It supports one-time and recurring schedules with timezone awareness, flexible time windows, configurable retries, and direct targeting of 270+ AWS APIs.

## Goal

Create a one-time schedule that publishes to SNS 5 minutes from now, then create a cron schedule for the same target. Show both forms.

## Prerequisites

- AWS CLI v2
- An SNS topic for the demo
- IAM permissions: `scheduler:*`, `iam:PassRole`, `sns:Publish`
- An IAM role assumable by `scheduler.amazonaws.com` with `sns:Publish` on the topic

## Steps

### 1. Setup target and role

```bash
REGION="us-east-1"
TOPIC_ARN=$(aws sns create-topic --name poc-scheduler-topic --region "$REGION" --query TopicArn --output text)

# Subscribe your email (confirm via inbox)
aws sns subscribe --topic-arn "$TOPIC_ARN" --protocol email --notification-endpoint you@example.com
```

Create role `scheduler-poc-role` trusting `scheduler.amazonaws.com` with an inline policy allowing `sns:Publish` on `$TOPIC_ARN`. Capture its ARN as `ROLE_ARN`.

### 2. One-time schedule (5 minutes from now)

```bash
WHEN=$(date -u -v+5M +'%Y-%m-%dT%H:%M:%S')   # macOS
# Linux: WHEN=$(date -u -d '+5 minutes' +'%Y-%m-%dT%H:%M:%S')

aws scheduler create-schedule \
  --name poc-once \
  --schedule-expression "at($WHEN)" \
  --schedule-expression-timezone "UTC" \
  --flexible-time-window '{"Mode":"OFF"}' \
  --target "{
      \"Arn\": \"$TOPIC_ARN\",
      \"RoleArn\": \"$ROLE_ARN\",
      \"Input\": \"Hello from EventBridge Scheduler one-time run\"
   }"
```

### 3. Recurring cron schedule

```bash
aws scheduler create-schedule \
  --name poc-cron \
  --schedule-expression "cron(*/10 * * * ? *)" \
  --schedule-expression-timezone "America/Sao_Paulo" \
  --flexible-time-window '{"Mode":"FLEXIBLE","MaximumWindowInMinutes":5}' \
  --target "{
      \"Arn\": \"$TOPIC_ARN\",
      \"RoleArn\": \"$ROLE_ARN\",
      \"Input\": \"Hello every 10 minutes\",
      \"RetryPolicy\": {\"MaximumRetryAttempts\": 3, \"MaximumEventAgeInSeconds\": 600}
   }"
```

### 4. Inspect

```bash
aws scheduler list-schedules
aws scheduler get-schedule --name poc-once
```

## Cleanup

```bash
aws scheduler delete-schedule --name poc-once
aws scheduler delete-schedule --name poc-cron
aws sns delete-topic --topic-arn "$TOPIC_ARN"
# Detach/delete the scheduler-poc-role
```

## Insight

EventBridge **Rules** (the old CloudWatch Events) are event-driven and limited to ~hundreds of rules per bus. EventBridge **Scheduler** is purpose-built for time-based invocation: millions of schedules per account, native timezone, flexible time window (to spread load), and per-target retry/DLQ config. It also targets nearly any AWS API directly (`arn:aws:scheduler:::aws-sdk:<service>:<action>`) without needing Lambda glue.

Use Scheduler instead of Rules whenever the trigger is a time, not an event.

## References

- Scheduler docs: https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html
- Universal targets: https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-universal.html
