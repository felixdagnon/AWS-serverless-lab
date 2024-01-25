# AWS-serverless-lab
Level up Lab for serverless

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/394f7201-1a03-4bde-bfb8-1f7d201c2cc1)

An Amazon API Gateway is a collection of resources and methods. For this tutorial, you create one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

Create, update, and delete an item.
Read an item.
Scan an item.
Other operations (echo, ping), not related to DynamoDB, that you can use for testing.
The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:
```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
```
The following is a sample request payload for a DynamoDB read item operation:
```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```
## Setup
### Create Lambda IAM Role 
Create the execution role that gives your function permission to access AWS resources.

To create an execution role

1. Open the roles page in the IAM console.
2. Choose Create role.
3. Create a role with the following properties.
    * Trusted entity – Lambda.
    * Role name – **lambda-apigateway-role**.
    * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that  the function needs t write data to DynamoDB and upload logs.
      
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "Stmt1428341300017",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "",
      "Resource": "*",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow"
    }
    ]
    }
    ```
### Create Lambda Function
**To create the function**
1. Click "Create function" in AWS Lambda Console

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/b63d5300-3b03-4d61-8efe-578f5870f596)

2. Select "Author from scratch". Use name **LambdaFunctionOverHttps** , select **Python 3.7** as Runtime. Under Permissions, select "Use an existing role", and select **lambda-apigateway-role** that we created, from the drop down

3. Click "Create function"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/0b9f210c-44cb-46ba-b22d-c8fefcf4c546)

4. Replace the boilerplate coding with the following code snippet and click "Save"

**Example Python Code*

```python
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```
![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/b3f4be8e-e3b9-4ff6-b6b9-219b643fc976)

### Test Lambda Function

Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.
1. Click the arrow on "Select a test event" and click "Configure test events"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/4f8e4075-d32f-4c52-8511-bd72a2220582)


2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Create" to save

```json
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
```
![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/312d15ae-b553-4f67-8016-5380ed99a67c)

3. Click "Test", and it will execute the test event. You should see the output in the console

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/bc1c6195-a055-4ceb-a175-b3c496dda319)

We're all set to create DynamoDB table and an API using our lambda as backend!

### Create DynamoDB Table

Create the DynamoDB table that the Lambda function uses.

**To create a DynamoDB table**

1. Open the DynamoDB console.
2. Choose Create table.
3. Create a table with the following settings.
   * Table name – lambda-apigateway
   * Primary key – id (string)
4. Choose Create.

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/96072470-2744-4b3c-b681-eec39b05eabf)

### Create API

**To create the API**
1. Go to API Gateway console
2. Click Create API

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/6d82fe51-3d45-47a6-8aa9-d9932ade9a13)

3. Scroll down and select "Build" for REST API

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/720d146a-2420-48fb-9788-483426b1f69b)

4. Give the API name as "DynamoDBOperations", keep everything as is, click "Create API"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/48bba13c-73c6-49bb-877f-9b3ae8518591)

5. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next 

Click "Actions", then click "Create Resource"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/f2ffaf95-7b26-40e5-a238-80f0fa9e4070)

6. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/4f7a6601-449b-4282-836e-5d560badbf9b)

7. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method". 

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/5f550e7d-dd38-4365-ab50-dd71f0c63bcc)

8. Select "POST" from drop down , then click checkmark

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/429bfb71-0316-4cf7-a337-5f69de90b2e9)

9. The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name, your function name will show up.Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/fce973b6-5447-409a-a83f-18c760340f5c)

Our API-Lambda integration is done!

### Deploy the API



