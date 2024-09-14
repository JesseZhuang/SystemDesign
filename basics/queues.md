Queues
====

- Queues are used to effectively manage requests in a large-scale distributed system, in which different components of the system may need to work in an asynchronous way.
- It is an abstraction between the client’s request and the actual work performed to service it.
- Queues are implemented on the asynchronous communication protocol. When a client submits a task to a queue they are no longer required to wait for the results
- Queue can provide protection from service outages and failures.

## Main Benefits

1. decouple
2. smooth traffic spikes
3. performance, async
4. granular scalability
5. improved reliability

## AWS SQS

SQS consumers generally polls from the queue. But with event bridge and trigger to launch Lambda, it can behave like pushing.

SQS triggers are not [free (but you knew that already)][1]. [SQS trigger support for Lambda][3] is implemented pretty much like you think it would be. The Lambda service [long-polls][2] your SQS queues for you, then triggers your Lambda function when messages appear. SQS API calls made by Lambda on your behalf are charged at the normal rate. Also be aware that Lambda will scale the number of pollers (minimum is 5) up or down based on demand.

Comparing SQS and SNS: [stackoverflow ref][4]

||SNS|SQS|
|-|-|-|
|persistence|no, lost after a few retries. intended for real-time applications|configurable 0-14 days|
|model|pub-sub|queue|

You don't have to couple SNS and SQS always. You can have SNS send messages to email, SMS or HTTP end point apart from SQS. There are advantages to coupling SNS with SQS. You may not want an external service to make connections to your hosts (a firewall may block all incoming connections to your host from outside).

Your end point may just die because of heavy volume of messages. Email and SMS maybe not your choice of processing messages quickly. By coupling SNS with SQS, you can receive messages at your pace. It allows clients to be offline, tolerant to network and host failures. You also achieve guaranteed delivery. If you configure SNS to send messages to an HTTP end point or email or SMS, several failures to send message may result in messages being dropped.


[1]: https://www.lucidchart.com/blog/cloud/5-reasons-why-sqs-lambda-triggers-are-a-big-deal#:~:text=SQS%20trigger%20support%20for%20Lambda,Lambda%20function%20when%20messages%20appear
[2]: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html
[3]: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-lambda-function-trigger.html
[4]: https://stackoverflow.com/questions/13681213/what-is-the-difference-between-amazon-sns-and-amazon-sqs
