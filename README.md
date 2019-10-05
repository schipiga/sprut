# Concurrent queue consumer

##### Service, which consumes messages from a queue in multithreading mode and pass them to handlers, launched with other threads as well.

This project is born under inspiration of https://github.com/goodmanship/sqsworkers/ , where I would like to implement my vision of proper architecture in such projects.

My main concern was to make consumer more abstract, not related with AWS SQS only.

But of course my personal usage for SQS.

With it's code it follows conveyor conception.

## How to use

```python
import boto3
from queue_consumer import Consumer


class Queue:

    def __init__(self,
                queue_name,
                max_number_of_messages=10,
                wait_time_seconds=20):
        self._max_number_of_messages = max_number_of_messages
        self._wait_time_seconds = wait_time_seconds
        self._sqs = boto3.resource("sqs").get_queue_by_name(
            QueueName=queue_name)


    def get(self):
        return self._sqs.receive_messages(
            AttributeNames=["All"],
            MessageAttributeNames=["All"],
            MaxNumberOfMessages=self._max_number_of_messages,
            WaitTimeSeconds=self._wait_time_seconds,
        )

    def cleanup(self, messages):
        self._sqs.delete_messages(
            Entries=[
                {
                    "Id": message.message_id,
                    "ReceiptHandle": message.receipt_handle,
                }
                for message in messages
            ]
        )

    def handler(self, messages):
        for message in messages:
            do_some_stuff(message)


consumer = Consumer(Queue("my_queue"))
consumer.start()
consumer.supervise(blocking=True)
```

if queue with get already exists, handler can be defined separately, like:

```python
def handler(messages):
    for message in messages:
        do_some_stuff(message)
```

A hanlder takes iterator as argument. If handler raises exception, worker defines not processed (failed) messages basing on iterator remaining content. That's why messages should be read & processed one-by-one. To read all iterator before processing is bad idea.

**Right:**

```python
def handler(message):
    for message in messages:
        process(message)
```

**Wrong:**

```python
def handler(messages):
    for message in list(messages):
        process(message)
```

It doesn't matter when bulk_size is 1 (default), when to read one message is the same as to read all iterator.
