# awslambda-toy-model
This is an example of a Serverless AI Data Engineering Pipeline. 
With CloudWatch Timer we will invoke a AWS Lambda function that reads a DynamoDB table and send each item as a message to a SQS queue
![68747470733a2f2f757365722d696d616765732e67697468756275736572636f6e74656e742e636f6d2f35383739322f35353335343438332d62616537616638302d353437612d313165392d393930392d6135363231323531303635622e706e67](https://user-images.githubusercontent.com/78228205/197618907-8681e765-4f53-4a0a-84e0-6aa785616b69.png)



# Create and Setup environment
1. Create a new Cloud9 environment *
2. Create a new SQS Queues service *
3. Create a new  DynamoDB table, and then create items inside that table *
4. Create a AWS ECR instance in which we will push our docker image *
5. Create and push docker image containing the app to ECR
6. Create AWS lambda function running in our docker image
7. Create a trigger with AWS EventBridge
8. Check if it works

.* This are steps that I will not explain or do it briefly
### 3. Create a new  DynamoDB table, and then create items inside that table
* The DynamoDB  table will be created with name 'fang' and it will have 5 items ('netflix', 'facebook', 'apple', 'google', 'wework')
![Screenshot from 2022-10-24 22-11-05](https://user-images.githubusercontent.com/78228205/197619690-1bb2cde3-cec0-44bd-8a8a-3d749de20921.png)


### 5. Create and push docker image containint the app to ECR

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
* Create the docker image and push to ECR following AWS info
![Screenshot from 2022-10-24 20-36-15](https://user-images.githubusercontent.com/78228205/197611300-48b2dce9-ce78-440d-aeab-b19ea8cc3821.png)
![Screenshot from 2022-10-24 20-49-09](https://user-images.githubusercontent.com/78228205/197611619-54427c0a-0427-427d-8bb7-d9a2e107aa6b.png)
![Screenshot from 2022-10-24 20-48-33](https://user-images.githubusercontent.com/78228205/197611940-5a48eed1-11d5-46f9-b30e-7b786c2636f3.png)
![Screenshot from 2022-10-24 20-49-48](https://user-images.githubusercontent.com/78228205/197612647-9bf98dfe-3593-447e-bec8-3fcb2e4e0401.png)
* Check ECR instance
![Screenshot from 2022-10-24 20-50-49](https://user-images.githubusercontent.com/78228205/197613065-21eb78b4-b72a-472f-8a90-9bf891d2e2ec.png)

### 6. Create AWS lambda function running in our docker image

* Crate AWS lambda function using a container image and make a test to see if it is working and logging correctly
![Screenshot from 2022-10-24 20-53-23](https://user-images.githubusercontent.com/78228205/197613485-37d441fa-8242-423a-8ba3-7fbfdd3122f1.png)
![Screenshot from 2022-10-24 20-57-05](https://user-images.githubusercontent.com/78228205/197613925-c5bc1198-1a6f-4a34-bd77-bccde271547e.png)

### 7. Create a trigger with AWS EventBridge
* With rate(1 minute) rule schedule
![Screenshot from 2022-10-24 20-57-45](https://user-images.githubusercontent.com/78228205/197615185-eb98726d-da39-4b40-82d5-8d40dc348c1d.png)
![Screenshot from 2022-10-24 21-02-17](https://user-images.githubusercontent.com/78228205/197615951-47bcd5f2-235f-487d-aae2-ae090ff438e3.png)

### 8. Check if it works
* After a couple of minutes we check if the trigger works, the lambda fonction is invoked, the AWS SQS service is receiving the queues, and if the load is correct.
![Screenshot from 2022-10-24 22-03-13](https://user-images.githubusercontent.com/78228205/197618255-360f2d71-8f97-4648-a247-4bfec7201869.png)
![Screenshot from 2022-10-24 22-00-29](https://user-images.githubusercontent.com/78228205/197617792-7f1e349a-30e7-46f2-8674-bd87f557e9fd.png)
![Screenshot from 2022-10-24 21-07-31](https://user-images.githubusercontent.com/78228205/197617069-e9455c08-694f-4247-8a29-3d4d067941e3.png)
![Screenshot from 2022-10-24 21-03-30](https://user-images.githubusercontent.com/78228205/197617327-ddcbe133-a04b-4e9a-ac28-436ffca09cd6.png)
![Screenshot from 2022-10-24 21-03-37](https://user-images.githubusercontent.com/78228205/197617374-ab5b2547-c968-462c-95f6-8f0ec6713def.png)

