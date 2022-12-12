Queues
====

- Queues are used to effectively manage requests in a large-scale distributed system, in which different components of the system may need to work in an asynchronous way.
- It is an abstraction between the client’s request and the actual work performed to service it.
- Queues are implemente on the asynchronious communication protocol. When a client submits a task to a queue they are no longer required to wait for the results
- Queue can provide protection from service outages and failures.

## AWS SQS

SQS consumers generally polls from the queue. But with event bridge and trigger to launch Lambda, it can behave like pushing.

SQS triggers are not free (but you knew that already). SQS trigger support for Lambda is implemented pretty much like you think it would be. The Lambda service long-polls your SQS queues for you, then triggers your Lambda function when messages appear. SQS API calls made by Lambda on your behalf are charged at the normal rate. Also be aware that Lambda will scale the number of pollers (minimum is 5) up or down based on demand. (https://www.lucidchart.com/blog/cloud/5-reasons-why-sqs-lambda-triggers-are-a-big-deal#:~:text=SQS%20trigger%20support%20for%20Lambda,Lambda%20function%20when%20messages%20appear.)

1. https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html
1. https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-lambda-function-trigger.html
