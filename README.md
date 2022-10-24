# awslambda-toy-model
This is an example of a Serverless AI Data Engineering Pipeline



# Create and Setup environment
1. Create a new Cloud9 environment *
2. Create a new SQS Queues service *
3. Create a new  DynamoDB table, and then create items inside that table *
4. Create a AWS ECR instance in which we will push our docker image *
5. Create and push docker image containing the app to ECR
5. Create AWS lambda function running in our docker image
6. Create a trigger with AWS EventBridge
7. Check if it works

.* This are steps that I will not explain


### Create and push docker image containint the app to ECR

* Create the docker directory and populate it with 'app.py', 'Dockerfile' and
    'requirements.txt'<br>
We have created a github repository and clone it in a EC2 instance that is where
I am working in '~/awslambda-toy-model'.
    ```bash
    mkdir lambda-docker \
    && touch app.py \
    && touch Dockerfile \
    && touch requirements.txt
    ```
* Populate 'app.py' <br>
    This will be a function that scan DynamoDB table and return the results
    sending a message for each item in the table.
    ```python
    """
    Dynamo to SQS
    """
    
    import boto3
    import json
    import sys
    import os
    
    DYNAMODB = boto3.resource('dynamodb')
    TABLE = "fang"
    QUEUE = "producer"
    SQS = boto3.client("sqs")
    
    # SETUP LOGGING
    import logging
    from pythonjsonlogger import jsonlogger
    
    LOG = logging.getLogger()
    LOG.setLevel(logging.INFO)
    logHandler = logging.StreamHandler()
    formatter = jsonlogger.JsonFormatter()
    logHandler.setFormatter(formatter)
    LOG.addHandler(logHandler)
    
    def scan_table(table):
        """Scans table and return results"""
    
        LOG.info(f"Scanning Table {table}")
        producer_table = DYNAMODB.Table(table)
        response = producer_table.scan()
        items = response['Items']
        LOG.info(f"Found {len(items)} Items")
        return items
    
    def send_sqs_msg(msg, queue_name, delay=0):
        """Send SQS Message
    
        Expects an SQS queue_name and msg in a dictionary format.
        Returns a response dictionary. 
        """
    
        queue_url = SQS.get_queue_url(QueueName=queue_name)["QueueUrl"]
        queue_send_log_msg = "Send message to queue url: %s, with body: %s" %\
            (queue_url, msg)
        LOG.info(queue_send_log_msg)
        json_msg = json.dumps(msg)
        response = SQS.send_message(
            QueueUrl=queue_url,
            MessageBody=json_msg,
            DelaySeconds=delay)
        queue_send_log_msg_resp = "Message Response: %s for queue url: %s" %\
            (response, queue_url) 
        LOG.info(queue_send_log_msg_resp)
        return response
    
    def send_emissions(table, queue_name):
        """Send Emissions"""
    
        items = scan_table(table=table)
        for item in items:
            LOG.info(f"Sending item {item} to queue: {queue_name}")
            response = send_sqs_msg(item, queue_name=queue_name)
            LOG.debug(response)
    
    def lambda_handler(event, context):
        """
        Lambda entrypoint
        """
    
        extra_logging = {"table": TABLE, "queue": QUEUE}
        LOG.info(f"event {event}, context {context}", extra=extra_logging)
        send_emissions(table=TABLE, queue_name=QUEUE)
    ```
* Populate 'Dockerfile' <br>
    ```
    # Python 3.8 base image, from "gallery.ecr.aws/"
    FROM public.ecr.aws/lambda/python:3.8
    
    # copy requirements to conainer
    COPY requirements.txt ./
    
    # Install dependencies
    RUN pip install -r requirements.txt
    
    # copy app.py to container
    COPY app.py ./
    
    # setting the CMD to the handler file_name.function_name
    CMD [ "app.lambda_handler" ]
    ```
* Populate 'requirements.txt' <br>
    ```
    python-json-logger
    boto3
    ```
* Create the docker image and push to ECR