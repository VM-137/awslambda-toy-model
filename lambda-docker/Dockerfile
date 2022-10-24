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