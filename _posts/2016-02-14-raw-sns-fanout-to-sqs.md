---
title: raw sns fanout to sqs
tags: aws sns sqs
---

The other day I was messing around a bit with AWS SNS and SQS, more specifically I
wanted to send messages to a SNS topic and have the messages fanout to a couple
of SQS queues (as mentioned in the [Common SNS Scenarios](https://docs.aws.amazon.com/sns/latest/dg/SNS_Scenarios.html)
documentation).

It's simple enough to connect a SNS topic with a SQS queue:

```shell
$ aws sns subscribe --topic-arn $SNS_TOPIC_ARN \
                    --protocol sqs \
                    --notification-endpoint $SQS_QUEUE_ARN
```

However, the messages sent from SNS to SQS end up on a format that's pretty far
from what I wanted:

```shell
$ aws sns publish --topic-arn $SNS_TOPIC_ARN --message '{"hello": "world"}'
$ aws sqs receive-message --queue-url $SQS_QUEUE_URL
{
  "Messages": [
    {
      "Body": "{\n  \"Type\" : \"Notification\",\n  \"MessageId\" : \"42e88f9d-6436-555e-aa9e-c0a7b128eeb8\",\n  \"TopicArn\" : \"arn:aws:sns:eu-west-1:123456789000:test\",\n  \"Message\" : \"{\\\"hello\\\": \\\"world\\\"}\",\n  \"Timestamp\" : \"2016-02-14T19:17:50.008Z\",\n  \"SignatureVersion\" : \"1\"\n}"
    }
  ]
}
```

I abbreviated the message a bit, but it's clear that the full SNS message has
been JSON encoded and put into the `Body` of the SQS message.

So what I really wanted was to simply have the messages forwarded from SNS to
SQS, without any surprises.
Turns out it's actually possible to achieve exactly that, though I had a hard
time finding it in the documentation (not exactly sure if I even found it in the
documentation).
The trick is to enable the `RawMessageDelivery` property of the SNS
subscription, for example using `aws`:

```shell
$ aws sns set-subscription-attributes --subscription-arn $SNS_SUBSCRIPTION_ARN \
                                      --attribute-name RawMessageDelivery \
                                      --attribute-value true
```
