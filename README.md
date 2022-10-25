# Serverless AI Data Engineering Pipeline
This is an example of a Serverless AI Data Engineering Pipeline.<br>
With CloudWatch Timer we will invoke a AWS Lambda function(Producer) that reads a DynamoDB table and send each item as a message to a SQS queue, when a message feeds the queue a trigger will invoke a AWS Lambda function(Consumer) that will perform a request to Wikipedia API and ask for the first result and the first 2 lines of the content, then we will perform a sentiment analysis using AWS Compehend and store the results in a AWS S3 bucket.

![68747470733a2f2f757365722d696d616765732e67697468756275736572636f6e74656e742e636f6d2f35383739322f35353335343438332d62616537616638302d353437612d313165392d393930392d6135363231323531303635622e706e67](https://user-images.githubusercontent.com/78228205/197618907-8681e765-4f53-4a0a-84e0-6aa785616b69.png)

*schema names, lambda 'producer' function will be called 'sqs-pipeline' and 'consumer' will be called 'sqs-pipeline2'

# Create and Setup environment
1. Create a new Cloud9 environment *
2. Create a new SQS Queues service *
3. Create a new  DynamoDB table, and then create items inside that table *
4. Create a AWS ECR repo 'dokcer-lambda-sqspipeline' for 'sqs-pipeline' (producer) lambda function *
5. Create and push docker image 'latest' to AWS ECR repo 'dokcer-lambda-sqspipeline'
6. Create AWS lambda 'sqs-pipeline' function running in our 'latest' docker image
7. Create a trigger with AWS EventBridge for 'sqs-pipeline' lambda function
8. Create a AWS ECR repo 'dokcer-lambda-sqspipeline2' for 'sqs-pipeline2' (consumer) lambda function *
9. Create and push docker image 'latest' to AWS ECR repo (dokcer-lambda-sqspipeline2)
10. Create AWS lambda 'sqs-pipeline2' function running in our 'latest' docker image *
11. Create a SQS trigger for 'sqs-pipeline' *
12. Check if it work correctly

.* This are steps that I will not explain or do it briefly
### 3. Create a new  DynamoDB table, and then create items inside that table
* The DynamoDB  table will be created with name 'fang' and it will have 5 items ('netflix', 'facebook', 'apple', 'google', 'wework')
![Screenshot from 2022-10-24 22-11-05](https://user-images.githubusercontent.com/78228205/197619690-1bb2cde3-cec0-44bd-8a8a-3d749de20921.png)


### 5. Create and push docker image 'latest' to AWS ECR repo 'dokcer-lambda-sqspipeline'

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

### 6. Create AWS lambda 'sqs-pipeline' function running in our 'latest' docker image

* Crate AWS lambda function using a container image and make a test to see if it is working and logging correctly
![Screenshot from 2022-10-24 20-53-23](https://user-images.githubusercontent.com/78228205/197613485-37d441fa-8242-423a-8ba3-7fbfdd3122f1.png)
![Screenshot from 2022-10-24 20-57-05](https://user-images.githubusercontent.com/78228205/197613925-c5bc1198-1a6f-4a34-bd77-bccde271547e.png)

### 7. Create a trigger with AWS EventBridge for 'sqs-pipeline' lambda function
* With rate(1 minute) rule schedule
![Screenshot from 2022-10-24 20-57-45](https://user-images.githubusercontent.com/78228205/197615185-eb98726d-da39-4b40-82d5-8d40dc348c1d.png)
![Screenshot from 2022-10-24 21-02-17](https://user-images.githubusercontent.com/78228205/197615951-47bcd5f2-235f-487d-aae2-ae090ff438e3.png)

### 9. Create and push docker image 'latest' to AWS ECR repo (dokcer-lambda-sqspipeline2)

* Create the docker directory and populate it with 'app.py', 'Dockerfile' and
    'requirements.txt'<br>
    ```bash
    mkdir lambda-docker2 \
    && touch app.py \
    && touch Dockerfile \
    && touch requirements.txt
    ```
* Populate 'app.py' <br>
    This is a script that allow us to:
    1. connect to AWS SQS service and read, delete or count the messages left on a queue.
    2. Send request to Wikipedia API
    3. Use AWS Comprehend to do sentiment analysis
    4. Send the results to a AWS S3 bucket
    ```python
    import json

    import boto3
    import botocore

    # import pandas as pd
    import pandas as pd
    import wikipedia
    import boto3
    from io import StringIO


    # SETUP LOGGING
    import logging
    from pythonjsonlogger import jsonlogger

    LOG = logging.getLogger()
    LOG.setLevel(logging.DEBUG)
    logHandler = logging.StreamHandler()
    formatter = jsonlogger.JsonFormatter()
    logHandler.setFormatter(formatter)
    LOG.addHandler(logHandler)

    # S3 BUCKET
    REGION = "us-east-1"

    ### SQS Utils###
    def sqs_queue_resource(queue_name):
        """Returns an SQS queue resource connection
        Usage example:
        In [2]: queue = sqs_queue_resource("dev-job-24910")
        In [4]: queue.attributes
        Out[4]:
        {'ApproximateNumberOfMessages': '0',
         'ApproximateNumberOfMessagesDelayed': '0',
         'ApproximateNumberOfMessagesNotVisible': '0',
         'CreatedTimestamp': '1476240132',
         'DelaySeconds': '0',
         'LastModifiedTimestamp': '1476240132',
         'MaximumMessageSize': '262144',
         'MessageRetentionPeriod': '345600',
         'QueueArn': 'arn:aws:sqs:us-west-2:414930948375:dev-job-24910',
         'ReceiveMessageWaitTimeSeconds': '0',
         'VisibilityTimeout': '120'}
        """

        sqs_resource = boto3.resource("sqs", region_name=REGION)
        log_sqs_resource_msg = (
            "Creating SQS resource conn with qname: [%s] in region: [%s]"
            % (queue_name, REGION)
        )
        LOG.info(log_sqs_resource_msg)
        queue = sqs_resource.get_queue_by_name(QueueName=queue_name)
        return queue


    def sqs_connection():
        """Creates an SQS Connection which defaults to global var REGION"""

        sqs_client = boto3.client("sqs", region_name=REGION)
        log_sqs_client_msg = "Creating SQS connection in Region: [%s]" % REGION
        LOG.info(log_sqs_client_msg)
        return sqs_client


    def sqs_approximate_count(queue_name):
        """Return an approximate count of messages left in queue"""

        queue = sqs_queue_resource(queue_name)
        attr = queue.attributes
        num_message = int(attr["ApproximateNumberOfMessages"])
        num_message_not_visible = int(attr["ApproximateNumberOfMessagesNotVisible"])
        queue_value = sum([num_message, num_message_not_visible])
        sum_msg = (
            """'ApproximateNumberOfMessages' and 'ApproximateNumberOfMessagesNotVisible' = *** [%s] *** for QUEUE NAME: [%s]"""
            % (queue_value, queue_name)
        )
        LOG.info(sum_msg)
        return queue_value


    def delete_sqs_msg(queue_name, receipt_handle):

        sqs_client = sqs_connection()
        try:
            queue_url = sqs_client.get_queue_url(QueueName=queue_name)["QueueUrl"]
            delete_log_msg = "Deleting msg with ReceiptHandle %s" % receipt_handle
            LOG.info(delete_log_msg)
            response = sqs_client.delete_message(
                QueueUrl=queue_url, ReceiptHandle=receipt_handle
            )
        except botocore.exceptions.ClientError as error:
            exception_msg = (
                "FAILURE TO DELETE SQS MSG: Queue Name [%s] with error: [%s]"
                % (queue_name, error)
            )
            LOG.exception(exception_msg)
            return None

        delete_log_msg_resp = "Response from delete from queue: %s" % response
        LOG.info(delete_log_msg_resp)
        return response


    def names_to_wikipedia(names):

        wikipedia_snippit = []
        for name in names:
            wikipedia_snippit.append(wikipedia.summary(name, sentences=2, auto_suggest=False))
        df = pd.DataFrame({"names": names, "wikipedia_snippit": wikipedia_snippit})
        return df


    def create_sentiment(row):
        """Uses AWS Comprehend to Create Sentiments on a DataFrame"""

        LOG.info(f"Processing {row}")
        comprehend = boto3.client(service_name="comprehend")
        payload = comprehend.detect_sentiment(Text=row, LanguageCode="en")
        LOG.debug(f"Found Sentiment: {payload}")
        sentiment = payload["Sentiment"]
        return sentiment


    def apply_sentiment(df, column="wikipedia_snippit"):
        """Uses Pandas Apply to Create Sentiment Analysis"""

        df["Sentiment"] = df[column].apply(create_sentiment)
        return df


    ### S3 ###


    def write_s3(df, bucket, name):
        """Write S3 Bucket"""

        csv_buffer = StringIO()
        df.to_csv(csv_buffer)
        s3_resource = boto3.resource("s3")
        filename = f"{name}_sentiment.csv"
        res = s3_resource.Object(bucket, filename).put(Body=csv_buffer.getvalue())
        LOG.info(f"result of write to bucket: {bucket} with:\n {res}")


    def lambda_handler(event, context):
        """Entry Point for Lambda"""

        LOG.info(f"SURVEYJOB LAMBDA, event {event}, context {context}")
        receipt_handle = event["Records"][0]["receiptHandle"]  # sqs message
        #'eventSourceARN': 'arn:aws:sqs:us-east-1:561744971673:producer'
        event_source_arn = event["Records"][0]["eventSourceARN"]

        names = []  # Captured from Queue

        # Process Queue
        for record in event["Records"]:
            body = json.loads(record["body"])
            company_name = body["name"]

            # Capture for processing
            names.append(company_name)

            extra_logging = {"body": body, "company_name": company_name}
            LOG.info(
                f"SQS CONSUMER LAMBDA, splitting sqs arn with value: {event_source_arn}",
                extra=extra_logging,
            )
            qname = event_source_arn.split(":")[-1]
            extra_logging["queue"] = qname
            LOG.info(
                f"Attemping Deleting SQS receiptHandle {receipt_handle} with queue_name {qname}",
                extra=extra_logging,
            )
            res = delete_sqs_msg(queue_name=qname, receipt_handle=receipt_handle)
            LOG.info(
                f"Deleted SQS receipt_handle {receipt_handle} with res {res}",
                extra=extra_logging,
            )

        # Make Pandas dataframe with wikipedia snippts
        LOG.info(f"Creating dataframe with values: {names}")
        df = names_to_wikipedia(names)

        # Perform Sentiment Analysis
        df = apply_sentiment(df)
        LOG.info(f"Sentiment from FANG companies: {df.to_dict()}")

        # Write result to S3
        write_s3(df=df, bucket="sqs-wiki-sentiment", name=names)
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
    pandas
    wikipedia
    ```
* Create the docker image and push to ECR following AWS info


### 12. Check if it works
* First check the 'producer' function and the trigger. We start both and after a couple of minutes we check if the trigger works, the lambda fonction is invoked, the AWS SQS service is receiving the queues, and if the load is correct.
![Screenshot from 2022-10-24 22-03-13](https://user-images.githubusercontent.com/78228205/197618255-360f2d71-8f97-4648-a247-4bfec7201869.png)
![Screenshot from 2022-10-24 22-00-29](https://user-images.githubusercontent.com/78228205/197617792-7f1e349a-30e7-46f2-8674-bd87f557e9fd.png)
![Screenshot from 2022-10-24 21-07-31](https://user-images.githubusercontent.com/78228205/197617069-e9455c08-694f-4247-8a29-3d4d067941e3.png)
![Screenshot from 2022-10-24 21-03-30](https://user-images.githubusercontent.com/78228205/197617327-ddcbe133-a04b-4e9a-ac28-436ffca09cd6.png)
![Screenshot from 2022-10-24 21-03-37](https://user-images.githubusercontent.com/78228205/197617374-ab5b2547-c968-462c-95f6-8f0ec6713def.png)

* Second check the 'consumer' function log and the S3 bucket objects
![Screenshot from 2022-10-25 02-34-10](https://user-images.githubusercontent.com/78228205/197655664-a3517cb2-2532-4d9b-96bc-eaaff7dd99cd.png)
![Screenshot from 2022-10-25 02-34-31](https://user-images.githubusercontent.com/78228205/197655678-f9ff3563-35e5-4084-ac2a-fc5aad6ee10f.png)
![Screenshot from 2022-10-25 02-36-19](https://user-images.githubusercontent.com/78228205/197655686-709c898a-115a-49f0-b08c-7965f9fa4dfb.png)
![Screenshot from 2022-10-25 02-36-47](https://user-images.githubusercontent.com/78228205/197655691-8da9c79e-228b-434d-a7a5-a5b3936a151b.png)

* All working


-I have followed [this](https://github.com/noahgift/awslambda)  repo
