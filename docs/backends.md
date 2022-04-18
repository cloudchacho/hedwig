# Backends

Hedwig works on cloud-native backends such as AWS and GCP. Before you can publish/consume messages, you need
to provision the required infra. This may be done manually, or, preferably, using Terraform. Hedwig provides tools to
make infra configuration easier: see [Terraform](https://www.terraform.io/) and
[Terraform Google](https://github.com/cloudchacho/terraform-google-hedwig) for further details.

## Fan Out

Hedwig utilizes [SNS](https://aws.amazon.com/sns/) and [Pub/Sub](https://cloud.google.com/pubsub/docs/overview) for
fan-out configuration. A publisher publishes messages on a [topic](/hedwig/concepts). These messages may be
received by zero or more consumers.

## Manual provisioning

At a high level, Hedwig requires one topic per message type, one queue and one dead letter queue per application and 
one subscription per-consumer-per-topic.

### GCP

In GCP, queues are called "subscriptions", and there's no support for automatic fan-in from multiple topics into a 
single subscription, and thus an app must subscribe to multiple subscriptions.

1. Install [gcloud](https://cloud.google.com/sdk/gcloud)
1. Authenticate with gcloud:

  ```shell
$ gcloud auth application-default login
  ```
1. Configure project (and repeat as appropriate for topics / apps):

  ```shell
$ APP=<myapp>
$ TOPIC=<mytopic>

$ gcloud config set project <GCP_PROJECT_ID>
$ gcloud pubsub topics create hedwig-${APP}-dlq
$ gcloud pubsub subscriptions create hedwig-${APP}-dlq --topic hedwig-${APP}-dlq
$ gcloud pubsub topics create hedwig-${TOPIC}
$ gcloud pubsub subscriptions create hedwig-${APP}-${TOPIC} --topic hedwig-${TOPIC} --dead-letter-topic hedwig-${APP}-dlq
$ gcloud pubsub topics create hedwig-${APP}
$ gcloud pubsub subscriptions create hedwig-${APP} --topic hedwig-${APP} --dead-letter-topic hedwig-${APP}-dlq
  ```

### AWS

1. Install [awscli](https://aws.amazon.com/cli/)
1. Authenticate with AWS:

  ```shell
$ aws configure
  ```
1. Configure project:

  ```shell
$ APP=<myapp>

$ AWS_REGION=$(aws configure get region)
$ AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
$ aws sns create-topic --name hedwig-${TOPIC}
$ aws sqs create-queue --queue-name HEDWIG-${APP}
$ aws sqs create-queue --queue-name HEDWIG-${APP}-DLQ
$ aws sns subscribe --topic-arn arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:hedwig-${TOPIC} --protocol sqs --notification-endpoint arn:aws:sqs:${AWS_REGION}:${AWS_ACCOUNT_ID}:HEDWIG-${APP} --attributes RawMessageDelivery=true
$ aws sqs set-queue-attributes --queue-url https://${AWS_REGION}.queue.amazonaws.com/${AWS_ACCOUNT_ID}/HEDWIG-${APP} --attributes "{\"Policy\":\"{\\\"Version\\\":\\\"2012-10-17\\\",\\\"Statement\\\":[{\\\"Action\\\":[\\\"sqs:SendMessage\\\",\\\"sqs:SendMessageBatch\\\"],\\\"Effect\\\":\\\"Allow\\\",\\\"Resource\\\":\\\"arn:aws:sqs:${AWS_REGION}:${AWS_ACCOUNT_ID}:HEDWIG-${APP}\\\",\\\"Principal\\\":{\\\"Service\\\":[\\\"sns.amazonaws.com\\\"]}}]}\",\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"arn:aws:sqs:${AWS_REGION}:${AWS_ACCOUNT_ID}:HEDWIG-${APP}-DLQ\\\",\\\"maxReceiveCount\\\":5}\"}"
  ```

## Infrastructure-as-code provisioning

Define a config file as such:
```json
{
    "queue_consumers": [
        {
            "queue": "<APP>",
            "tags": {
                "App": "<APP>",
                "Env": "<ENVIRONMENT>"
            },
            "subscriptions": [
                {
                    "topic": "<TOPIC>"
                }
            ]
        }
    ],
    "topics": [
       "<TOPIC>"
    ]
}
```

And generate terraform config using:
```shell
$ hedwig-terraform-generator config.json
```
