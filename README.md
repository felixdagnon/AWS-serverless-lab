# AWS serverless-lab
Level up Lab for serverless

Introduction

Let's see these 2 diagrams considering create website:

![image](https://github.com/felixdagnon/Serverless-lab/assets/91665833/8f9ba5b1-7e7c-4aaa-aa18-97d496d421a8)


Before doing the serverless microservice project, we will make the differences between non serverless and serverless architecture.

In our case non serverless (ALB, multiAZ,autoscalling group, EC2,MySQL RDS) and serverless (API Gateway, lambda, DynamoDB) architecture.

- The fisrt diagram We have to create Ec2 and even when there is no traffic to our website, this is it is still up and running, 

so we still need to pay for these ec2 even the utilization is very low.

A similarly for RDS, we needed to select EC2 instance type.And even when there is no utilization, we still need to pay for this instance 

as well as the secondary instance that's created because of the multi AC database.

Another thing is you needed to define the auto scaling group and you have to define when to scale out and all that stuff.

- Now the serverless design is much simpler and a lot of the features is provided out of the box.

Instead of application load balancer,  we are going to use API gateway and the back end of the business logic runs on Lambda.

So the big difference between EC2 and lambda is lambda will scale automatically if there are a lot of traffic coming in.

Also API Gateway is a fully managed API gateway service will scale automatically like application load balancer .

We don't need to do anything.

Another difference is for ec2 running the Web server in the first diagram, we have to go define the auto scaling group.

If we have not defined it, we won't be able to scale and you will get error. With lambda we do not need to go define

any auto scaling group. If there are multiple traffic, this lambda will scale up automatically and handle that traffic.

Another advantage Lambda is highly available. So even when one availability zone goes down, lambda will be up and running.

- Now the underlying database will be Dynamo.

So DynamoDB is a double OSS flagship, no SQL database compared to RDS, which is a relational database.

And the difference here is for RDS, we madke it multi AZ by clicking option, we pay for it but the failover is automatic.

But with DynamoDB it is inherently highly available. We don't need to go select anything.

All the data in the DynamoDB is automatically replicated to multiple availability zone.

The Second difference, for the relational database if the traffic grows like crazy, we need to create a read replica 

where the read traffic should read from read replica, then we need to change the code and everything because read replica gives you a separate url.

With dynamodb. we don't need to do any of these things. Highly available, scalable and pay as you go.

So if there is no traffic, we don't pay for lambda  and for DynamoDB If there is no traffic, we do not pay for any traffic, we only pay for the data storage.

Whereas for MySQL RDS we pay a fixed cost even if there is no traffic.

Now that we have the architecture explained, let's jump into the console, deploy this architecture


![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/4bfee298-e986-46cd-8b34-7c0e7cb5057b)


An Amazon API Gateway is a collection of resources and methods. For this demo, we create one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a 

Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

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

2. Select "Author from scratch". Use name **LambdaFunctionOverHttps** , select **Python 3.7** as Runtime. Under Permissions, select "Use an existing role", and select **lambda-

   apigateway-role** that we created, from the drop down

4. Click "Create function"

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


2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as

   output. Click "Create" to save

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

5. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a

resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next 

Click "Actions", then click "Create Resource"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/f2ffaf95-7b26-40e5-a238-80f0fa9e4070)

6. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/4f7a6601-449b-4282-836e-5d560badbf9b)

7. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method". 

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/5f550e7d-dd38-4365-ab50-dd71f0c63bcc)

8. Select "POST" from drop down , then click checkmark

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/429bfb71-0316-4cf7-a337-5f69de90b2e9)

9. The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name,

your function name will show up.Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/fce973b6-5447-409a-a83f-18c760340f5c)

Our API-Lambda integration is done!

### Deploy the API

In this step, you deploy the API that you created to a stage called prod.

1. Click "Actions", select "Deploy API"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/46aa17fe-c86c-45b3-9f72-537b94d99514)

2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/0e8670fc-bf11-4801-a2b5-54a6beed51fa)

3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/a48aac11-b305-4c78-903f-848aea439aea)

### Running our solution

1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```
2. To execute our API from local machine, we are going to use Postman and Curl command. You can choose either method based on your convenience and familiarity.
   
    * To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should

execute and return "HTTPStatusCode" 200.

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/b87fad28-9125-4ff9-8ae7-9152f5a814c1)

  * To run this from terminal using Curl, run the below
    ```
    $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
    ```   
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table, select "Items" tab, and the newly

inserted item should be displayed.

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/2ac4a95a-d342-481f-aed3-4d0b461b381c)

4. To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will

return all the items from the Dynamo table

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/21474b21-71e5-4cfe-b064-e06d4f879665)

5. The Lambda function supports using the create operation to update an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "update",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1234ABCD"
        },
        "AttributeUpdates": {
            "number": {
                "Value": 10
            }
        }
    }
}
```

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/62fe43be-4b1e-47c0-a7eb-6ecf30d186c9)

6. The Lambda function supports using the create operation to delete an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "delete",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1235ABCD"
        }
    }
}
```

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/7e037214-de23-45e7-a1f2-497657ea4728)

7. The Lambda function supports using the create operation to read an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1237ABCD"
        }
    }
}
```
![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/d84c56e2-d27a-4d7d-b0b7-c97d09f84f3b)

![image](https://github.com/felixdagnon/AWS-serverless-lab/assets/91665833/419e26ba-24d5-4cac-9ceb-69f956113676)






We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Cleanup

Let's clean up the resources we have created for this lab.


### Cleaning up DynamoDB

To delete the table, from DynamoDB console, select the table "lambda-apigateway", and click "Delete table"

![image](https://github.com/felixdagnon/Serverless-lab/assets/91665833/76c54e4a-54ee-4e97-9df6-49d98476121f)

To delete the Lambda, from the Lambda console, select lambda "LambdaFunctionOverHttps", click "Actions", then click Delete 

![image](https://github.com/felixdagnon/Serverless-lab/assets/91665833/93788cf8-1cbf-4beb-a688-c23cb39fefd4)

To delete the API we created, in API gateway console, under APIs, select "DynamoDBOperations" API, click "Actions", then "Delete"

![image](https://github.com/felixdagnon/Serverless-lab/assets/91665833/c856d48a-599b-43a9-8cb2-348f0bff5eac)







