---
layout: post
title: Data Continuity Service for DynamoDB
banner: /assets/posts/2018-06-13-Serverless-Basic-Authentication-Custom-Authorizer/banner.png
author: tvb
---

In this post I am going to explain how to build and configure a Data Continuity Service (DCS) for Amazon DynamoDB by using Amazon DynamoDB Streams, AWS Lambda and Amazon S3 services.

Why Data Continuity Service?
-----
Now you might ask yourself why you would need such thing as DCS because AWS launched the [backup and restore](https://aws.amazon.com/about-aws/whats-new/2017/11/aws-launches-amazon-dynamodb-backup-and-restore/) functionality for DynamoDB in November 2017.
Well, while this is true and it works really great there are some shortcomings to this feature.

To start with I would really like to recommend using the backup and restore functionality integrated with Amazon DynamoDB always for quick backup and recovery but it also means your backups are still hosted in the same region and within the same account your DynamoDB data is located.

Although highly unlikely, it could happen so that a region becomes out of service and you won't be able to access your DynamoDB data. Imagine your backups are in that same region. You would have no means to restore the service to your customers by restoring your DynamoDB tables because you are unable to access the data and the backups holding your precious data! It would mean disaster! And to make things even worse, imagine someone gets hold of your administrator access keys and starts deleting all of your AWS resources, data __and__ backups. [It would mean the end of your business](https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/) (must read!). This is where a Data Continuity Service for DynamoDB would be a perfect fit. The DCS I am going to show you will make sure your DynamoDB data is replicated to another region and into another account so you will always be able to access your data in case disaster really strikes.

Overview
--------
![overview](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/solution-overview.png)


Definitions
--------
**Amazon DynamoDB** Amazon DynamoDB is a fully managed proprietary NoSQL database service that supports key-value and document data structures.

**Amazon DynamoDB Streams** Amazon DynamoDB Streams captures a time-ordered sequence of item-level modifications in any DynamoDB table, and stores this information in a log for up to 24 hours.

**AWS Lambda** AWS Lambda is an event-driven, serverless computing platform. Run code without provisioning or managing servers.

**Amazon S3** Amazon S3 provides object storage at scale.


DynamoDB
-----------
For this post we are going to have a DynamoDB table called `users` which holds our company customers first name, last name, email address and an important field. Just for the sake of example.

It might look like the following:

![DynamoDB users table](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-users-table.png)

Here is a snippet of the CloudFormation template to setup the DynamoDB table:

```json
{
  "DynamoDbUsersTable": {
    "Properties": {
      "AttributeDefinitions": [
        {
          "AttributeName": "id",
          "AttributeType": "S"
        },
        {
          "AttributeName": "email",
          "AttributeType": "S"
        }
      ],
      "KeySchema": [
        {
          "AttributeName": "id",
          "KeyType": "HASH"
        }
      ],
      "TableName": "users"
    },
    "Type": "AWS::DynamoDB::Table"
  }
}
```


DynamoDB Streams
-----------
Now to get things going we need to configure a DynamoDB Stream on our `users` table so we can listen to the stream and trigger an action whenever a record in the table is changed.

>To read more about DynamoDB Streams see the [AWS documentation on DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html).

Here is a snippet to configure a stream in CloudFormation:

```json
{
  "DynamoDbUsersTable": {
    "Properties": {
      "..."
      "StreamSpecification": {
        "StreamViewType": "NEW_AND_OLD_IMAGES"
      },
      "TableName": "users"
    },
    "Type": "AWS::DynamoDB::Table"
  }
}
```

Note I stripped some of the other properties needed for easier reading. The real magic is the property called `StreamSpecification` and which the key "StreamViewType" is set to the string __NEW_AND_OLD_IMAGES__. This is _crucial_.
>To read more about the CloudFormation properties available for your DynamoDB table see the [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html#cfn-dynamodb-table-streamspecification).

Once we have configured our DynamoDB Stream, the Stream details on the DynamoDB overview page for the users table might look like the following:

![dynamodb-streams-enabled](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-streams-enabled.png)

We now have an Amazon Resource Name (ARN) available for us to use!

Lambda
--------
Ok so far it was easy going, right? Now lets move on to the part where the magic starts. AWS Lambda! I really like Lambda as it is so powerful yet so flexible.

We are going to use Lambda to listen to the DynamoDB Stream and whenever there is an event posted onto the stream Lambda will read the data on the stream and write it to Amazon S3.

To configure Lambda we first need to create an IAM Service Role. Here is the CloudFormation snippet:

```json
{
  "DynamoDbStreamToS3LambdaRole": {
    "Properties": {
      "AssumeRolePolicyDocument": {
        "Statement": [
          {
            "Action": [
              "sts:AssumeRole"
            ],
            "Effect": "Allow",
            "Principal": {
              "Service": "lambda.amazonaws.com"
            }
          }
        ],
        "Version": "2012-10-17"
      },
      "Path": "/",
      "Policies": [
        {
          "PolicyDocument": {
            "Statement": [
              {
                "Action": [
                  "logs:CreateLogGroup",
                  "logs:CreateLogStream",
                  "logs:DeleteSubscriptionFilter",
                  "logs:PutLogEvents",
                  "logs:PutSubscriptionFilter",
                  "logs:TestMetricFilter"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:logs:<YOUR_AWS_REGION>:<YOUR_AWS_ACCOUNT_ID>:*"
              }
            ]
          },
          "PolicyName": "AllowLogs"
        },
        {
          "PolicyDocument": {
            "Statement": [
              {
                "Action": [
                  "lambda:InvokeFunction"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:lambda:<YOUR_AWS_REGION>:<YOUR_AWS_ACCOUNT_ID>:function:*"
              }
            ]
          },
          "PolicyName": "AllowLambaInvoke"
        },
        {
          "PolicyDocument": {
            "Statement": [
              {
                "Action": [
                  "dynamodb:GetRecords",
                  "dynamodb:GetShardIterator",
                  "dynamodb:DescribeStream",
                  "dynamodb:ListStreams"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:dynamodb:<YOUR_AWS_REGION>:<YOUR_AWS_ACCOUNT_ID>:table/users/stream/*"
              }
            ]
          },
          "PolicyName": "AllowDynamoDbAccess"
        }
      ]
    },
    "Type": "AWS::IAM::Role"
  }
}
```

A few things are happening here. First we are allowing this role to assume on behalf of AWS Lambda by specifying the `sts:AssumeRole` permissions under the "AssumeRolePolicyDocument" key.

Next we are defining some needed policies. We are allowing Lambda to write to CloudWatch logs for logging. Next we allow Lambda to invoke the function or else it wouldn't start. And lastly we allowing Lambda to access and read the DynamoDB stream which is the hole point of this setup :smiley:

Once that is in place we can create our Lambda function and assign it our newly created execution role.

![lambda-function](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/lambda-function.png)

Or use the snippet below:

```json
{
  "DynamoDbStreamToS3LambdaFunction": {
    "Properties": {
      "Code": {
        "S3Bucket": "your-lambda-code-bucket",
        "S3Key": "dynamodb-stream-to-s3.zip"
      },
      "Environment": {
        "Variables": {
          "TABLE_NAME": "users"
        }
      },
      "Handler": "index.backup",
      "MemorySize": 128,
      "Role": {
        "Fn::GetAtt": [
          "DynamoDbStreamToS3LambdaRole",
          "Arn"
        ]
      },
      "Runtime": "nodejs6.10",
      "Timeout": 300
    },
    "Type": "AWS::Lambda::Function"
  }
}
```

After the function is created we need to attach the DynamoDB stream to our Lambda function so the function can listen and work whenever there is an event posted:

![configure lambda 1](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/configure-lambda-trigger-1.png)

Resulting in:

![configure lambda 2](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/configure-lambda-trigger-2.png)

Or use the snippet below:
```json
{
  "DynamoDbStreamToS3LambdaEventSourceMapping": {
    "Properties": {
      "EventSourceArn": "arn:aws:dynamodb:<YOUR_AWS_REGION>:<YOUR_AWS_ACCOUNT_ID>:table/users/stream/<STREAM_ID>",
      "FunctionName": {
        "Ref": "DynamoDbStreamToS3LambdaFunction"
      },
      "StartingPosition": "LATEST"
    },
    "Type": "AWS::Lambda::EventSourceMapping"
  }
}
```

Good. Everything on the Lambda side is ready apart from the Lambda function itself. Luckily the function itself is rather simple:

```javascript
'use strict';

const aws = require('aws-sdk');

const s3 = new aws.S3({ region: '<YOUR AWS_REGION>' });

module.exports.backup = (event, context, callback) => {
  const records = event.Records;

  Promise.all(records.map((record) => {
    const keysList = Object.keys(record.dynamodb.Keys).map((key) => {
      const keyDefinition = record.dynamodb.Keys[key];
      const type = Object.keys(keyDefinition)[0];
      const value = keyDefinition[type];
      return value;
    });

    const keysString = keysList.join('/');
    let image = {}
    if(typeof record.dynamodb.NewImage !== 'undefined') {
      image = aws.DynamoDB.Converter.output({ M: record.dynamodb.NewImage });
    }

    return s3.putObject({
      Bucket: process.env.BUCKET,
      Key: `${process.env.TABLE_NAME}/${keysString}/image.json`,
      Body: JSON.stringify(image),
      StorageClass: "STANDARD_IA"
    }).promise()
      .then((response) => {
        console.log(`${keysString} snapshot done`, response);
      })
      .catch((err) => {
        console.error('Error', err);
      });
  }))
    .then(v => callback(null, v), callback);
};
```

Make sure you are setting the Lambda handler to `index.backup`.

In general the function will for each DynamoDB __insert__ or __update__ action read the event record and write the "NewImage" data as a JSON file to S3 and store it as the _STANDARD_IA_ storage class to save up some bucks as you most probably never have to use the data. Or as least that is what we are hoping for. :sweat_smile:

It also expects an environment variable named `TABLE_NAME` which holds the DynamoDB table name. In this case the table is called `users` so we are sending that with the function. It also expects another environment variable named `BUCKET`. Ahhh finally! We haven't talked about buckets and S3 at all up to now so this would be a good moment to start talking about S3.

S3
-------
