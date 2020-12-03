# Serverless Architecture

Serverless does not mean that are no servers, but rather, refers to a server that will spin up only when required, and shut down when the job is done. Such an architecture has potential to save a lot costs.

Of course, the requirement for serverless is that the job should be something that is light, and not required to run 24/7.

This can be an AI microservice, or intermediate functions, e.g. taking data from S3 > process the data > send to AI microservice.

<hr>

## Lambda & API Gateway

In AWS, the serverless architecture is done using a lambda function, with an accompanying API gateway (if required), which generates an API URL for the function. The gateway can also control access through an API token, and have rate limits.

It is also important to note that the lambda function can only have a size of __50 Mb__, If required, certain blobs can be stored in S3 and lambda can access from there. It also have a default runtime of 30sec, and can be increase to a maximum of __15 mins__.

<hr>

## Zappa

[Zappa](https://github.com/Miserlou/Zappa) is a popular python library used to automatically launch python lambda functions & api-gateways.

### VENV

Your application/function needs a virtual environment with all the relevant python libraries installed in the root. See the [Virtual Environment](https://mapattacker.github.io/ai-engineer/virtual_env/) example on how to use venv for this. Before zappa deployment, ensure that the venv is activated and the app works.

### Setup AWS Policy & Keys for Zappa Deployment

The most difficult part of Zappa is to setup a user & policies for zappa to deploy your lambda/gateway application. First, we need to create a new user in AWS, and then define a policy which allows zappa to do all the work for deployment. Then we create the AWS access & secret keys, and port it in our local environment variables.

A healthy discussion on various minimium policies can be viewed [here](https://github.com/Miserlou/Zappa/issues/244). A working liberal example is given below.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "lambda:CreateFunction",
                "lambda:ListVersionsByFunction",
                "lambda:DeleteFunction",
                "lambda:GetFunctionConfiguration",
                "lambda:GetAlias",
                "lambda:InvokeFunction",
                "lambda:GetFunction",
                "lambda:UpdateFunctionConfiguration",
                "lambda:RemovePermission",
                "lambda:GetPolicy",
                "lambda:AddPermission",
                "lambda:DeleteFunctionConcurrency",
                "lambda:UpdateFunctionCode",
                "events:PutRule",
                "events:ListRuleNamesByTarget",
                "events:ListRules",
                "events:RemoveTargets",
                "events:ListTargetsByRule",
                "events:DescribeRule",
                "events:DeleteRule",
                "events:PutTargets",
                "logs:DescribeLogStreams",
                "logs:FilterLogEvents",
                "logs:DeleteLogGroup",
                "apigateway:DELETE",
                "apigateway:PATCH",
                "apigateway:GET",
                "apigateway:PUT",
                "apigateway:POST",
                "cloudformation:DescribeStackResource",
                "cloudformation:UpdateStack",
                "cloudformation:ListStackResources",
                "cloudformation:DescribeStacks",
                "cloudformation:CreateStack",
                "cloudformation:DeleteStack"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "iam:GetRole",
                "iam:PassRole",
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "s3:ListBucketMultipartUploads",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:iam::1234567890:role/*-ZappaLambdaExecutionRole",
                "arn:aws:s3:::<python-serverless-deployment-s3>"
            ]
        },
        {
            "Sid": "VisualEditor2",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": "arn:aws:s3:::<python-serverless-deployment-s3>/*"
        }
    ]
}
```

### How to Use

After installing zappa, we init it at the root of your function, which will create a default zappa_settings.json. 

```bash
pip install zappa
zappa init
```

All the instructions to launch the Lambda & API Gateway are set in `zappa_settings.json`. Refer to the official [readme](https://github.com/Miserlou/Zappa) for the entire list. We can have another stage, e.g. "prod" with its own configuration.

```json
{
    "dev": {
        "app_function": "app.app",
        "aws_region": "ap-southeast-1",
        "profile_name": "default",
        "project_name": "maskdetection-lambda",
        "runtime": "python3.6",
        "s3_bucket": "<python-serverless-deployment-s3>",
        "slim_handler": true,
        "apigateway_description": "lambda for AI microservice",
        "lambda_description": "lambda for AI microservice",
        "timeout_seconds": 60
    }
}
```

Then, we will deploy it to the cloud, by stating the stage. Behind the scenes, it does all these:

 * New user policy is created for lambda execution
 * Zappa will do some wrapping of your code & libraries, then package it as a zip
 * The zip will be uploaded to an S3 bucket stated in zappa_settings.json
 * Zip file is deleted
 * Lambda will recieve the zip from S3 and launch the application
 * API gateway to this lambda is setup

We might encounter errors along the way; to view the logs, we can use `zappa tail`, which extract the tail logs from AWS CloudWatch.

If we are using other frameworks to upload our lambda, e.g. [Serverless](https://www.serverless.com), we can also use zappa to zip the code only.

### Commands

| Cmd | Desc |
|-|-|
| zappa init | create a `zappa_settings.json` file |
| zappa deploy <stage> | deploy stage as specified in json |
| zappa update <stage> | update stage |
| zappa undeploy <stage> | delete lambda & API Gateway |
| zappa package <stage> | zip all files together |
| zappa tail <stage> | print tail logs from CloudWatch |
| zappa status | check status |


### Lambda Execution Role

The default execution role created by zappa is too liberal. From the author "It grants access to all actions for all resources for types CloudWatch, S3, Kinesis, SNS, SQS, DynamoDB, and Route53; lambda:InvokeFunction for all Lambda resources; Put to all X-Ray resources; and all Network Interface operations to all EC2 resources".

To set a manual policy, we need to set the "manage_roles" to `false` and include either the "role_name" or "role_arn". We then create the role & policies in AWS. 

```json
{
    "dev": {
        ...
        "manage_roles": false, // Disable Zappa client managing roles.
        "role_name": "MyLambdaRole", // Name of your Zappa execution role. Optional, default: <project_name>-<env>-ZappaExecutionRole.
        "role_arn": "arn:aws:iam::12345:role/app-ZappaLambdaExecutionRole", // ARN of your Zappa execution role. Optional.
        ...
    },
    ...
}
```

The trick is to first generate the default role, then to filter down the default policy to your specific usecase. Below is an example, which retrieves & updates S3 & DynamnoDB.


```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LambdaInvoke",
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:ap-southeast-1:123456780:function:complianceai-lambda-dev"
        },
        {
            "Sid": "Logs",
            "Effect": "Allow",
            "Action": "logs:*",
            "Resource": "arn:aws:logs:ap-southeast-1:123456780:log-group:/aws/lambda/complianceai-lambda-dev:*"
        },
        {
            "Sid": "S3ListObjectsInBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::compliance-bucket"
        },
        {
            "Sid": "S3AllObjectActions",
            "Effect": "Allow",
            "Action": "s3:*Object",
            "Resource": "arn:aws:s3:::compliance-bucket/*"
        },
        {
            "Sid": "DynamoDBListAndDescribe",
            "Effect": "Allow",
            "Action": [
                "dynamodb:List*",
                "dynamodb:DescribeReservedCapacity*",
                "dynamodb:DescribeLimits",
                "dynamodb:DescribeTimeToLive"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DynamoDBSpecificTable",
            "Effect": "Allow",
            "Action": [
                "dynamodb:BatchGet*",
                "dynamodb:DescribeStream",
                "dynamodb:DescribeTable",
                "dynamodb:Get*",
                "dynamodb:Query",
                "dynamodb:Scan",
                "dynamodb:BatchWrite*",
                "dynamodb:CreateTable",
                "dynamodb:Delete*",
                "dynamodb:Update*",
                "dynamodb:PutItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/ComplianceTable"
        }
    ]
}
```