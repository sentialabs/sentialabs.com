---
layout: post
title: Data Continuity Service for DynamoDB
banner: /assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/solution-overview.png)
author: tvb
---

This is the first post of three where I am going to explain how to build and configure a Data Continuity Service (DCS) for Amazon DynamoDB by using Amazon DynamoDB Streams, AWS Lambda and the Amazon S3 services.

Why Data Continuity Service?
-----
Now, you might ask yourself why you would need such thing as Data Continuity Service (DCS) for DynamoDB because AWS launched the [backup and restore](https://aws.amazon.com/about-aws/whats-new/2017/11/aws-launches-amazon-dynamodb-backup-and-restore/) functionality for DynamoDB in November 2017.
Well, while this is true and it works really great there are some shortcomings to the backup and restore functionality. Which I will explain now.

First, don't get me wrong! You should be using the backup and restore functionality integrated with Amazon DynamoDB always as it is really great at quickly making backup and restore your content. However it also means these backups are hosted in the same region and within the same account as your DynamoDB data is located.

So although, highly unlikely, it could happen that a region becomes out of service and you won't be able to access your DynamoDB data. Now imagine your backups are in that same region. You would have no means to restore the service to your customers by restoring your DynamoDB tables because you are unable to access the data and the backups holding your precious data! It would mean disaster! And to make things even worse, imagine someone gets hold of your administrator access keys and starts deleting all of your AWS resources, data __and__ backups. [It would mean the end of your business](https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/) (no joke, must read!). This is where a Data Continuity Service for DynamoDB would be a perfect fit. The DCS solution I am going to show you will make sure your DynamoDB data is replicated into another region and to another account so you will always be able to access your data in case disaster really strikes.

Overview
--------
![overview](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/solution-overview.png)


Definitions
--------
**Amazon DynamoDB** - Amazon DynamoDB is a fully managed proprietary NoSQL database service that supports key-value and document data structures.

**Amazon DynamoDB Streams** - Amazon DynamoDB Streams captures a time-ordered sequence of item-level modifications in any DynamoDB table, and stores this information in a log for up to 24 hours.

**AWS Lambda** - AWS Lambda is an event-driven, serverless computing platform. Run code without provisioning or managing servers.

**Amazon S3** - Amazon S3 provides object storage at scale.


DynamoDB
-----------
For this post we are going to have a DynamoDB table called `users` which holds our company customers first name, last name, email address and an important field. Just for the sake of example.

It might look like the following:

![DynamoDB users table](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-users-table.png)

Here is a snippet of a JSON formatted CloudFormation template to setup the DynamoDB table with CloudFormation:

```json
{
  "DynamoDbUsersTable": {
    "Properties": {
      "AttributeDefinitions": [
        {
          "AttributeName": "id",
          "AttributeType": "S"
        }
      ],
      "KeySchema": [
        {
          "AttributeName": "id",
          "KeyType": "HASH"
        }
      ],
      "ProvisionedThroughput": {
        "ReadCapacityUnits": 1,
        "WriteCapacityUnits": 1
      },
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

Note I stripped some of the other properties needed for easier reading. The real magic is the property called `StreamSpecification` and which holds the key "StreamViewType" and the value as a the string __NEW_AND_OLD_IMAGES__. This is _crucial_.
>To read more about the CloudFormation properties available for your DynamoDB table see the [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html#cfn-dynamodb-table-streamspecification).

Once we have configured our DynamoDB Stream, the Stream details on the DynamoDB overview page for the users table might look like the following:

![dynamodb-streams-enabled](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-streams-enabled.png)

We now have an Amazon Resource Name (ARN) available for us to use!

Lambda
--------
Ok so far it was easy going, right? We created a DynamoDB table and configured it to setup a DynamoDB Stream. Now lets move on to the part where the magic starts. AWS Lambda! I really like Lambda as it is so powerful yet so flexible.

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
                  "s3:PutObject"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:s3:::source-account-dynamodb-bucket/*"
              }
            ]
          },
          "PolicyName": "AllowS3Access"
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

A few things to note. First we are allowing this role to assume on behalf of AWS Lambda by specifying the `sts:AssumeRole` permissions under the "AssumeRolePolicyDocument" key.

Next we are defining some needed policies. We are allowing Lambda to write to CloudWatch logs for logging purposes. We also allowing Lambda to invoke the function or else it wouldn't be able start at all. We are allowing Lambda to write to the source bucket and lastly we allowing Lambda to access and read the DynamoDB Stream which is the hole point of this setup :smiley:

Don't forget to enter the correct AWS Region and your own AWS Account Id.

Once that is in place we can create our Lambda function and assign it our newly created execution role.

![lambda-function](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/lambda-function.png)

Or use the CloudFormation snippet below:

```json
{
  "DynamoDbStreamToS3LambdaFunction": {
    "Properties": {
      "Code": {
        "ZipFile": "exports.backup = function (event, context, callback){ console.log('Hello'); callback(null); }"
      },
      "Environment": {
        "Variables": {
          "BUCKET": "source-account-dynamodb-backup",
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

After the function is created we need to attach the DynamoDB Stream to our Lambda function so the function can listen and work whenever there is an event posted:

![configure lambda 1](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/configure-lambda-trigger-1.png)

Resulting in:

![configure lambda 2](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/configure-lambda-trigger-2.png)

Or update your CloudFormation template with the content below below:
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

Deployed? Done? Good. Everything on the Lambda side is ready apart from the function itself. Luckily the function itself is rather simple:

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

    const keysString = keysList[0];
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

Update the Lambda function with above code. Make sure you have set the Lambda handler to `index.backup`.

In general the function will run for each DynamoDB __insert__ or __update__ action and read the event record. It will then write the "NewImage" data as a JSON file to S3 and store it as the *STANDARD_IA* storage class to save up some bucks as you most probably never have to use the data. Or as least that is what we are hoping for. :sweat_smile:

It expects an environment variable named `TABLE_NAME` which holds the DynamoDB table name. In this case the table is called `users` so we are sending that with the function. It also expects another environment variable named `BUCKET`. Ahhh finally! We haven't really talked about buckets and S3 up to now so this would be a good moment to start talking about S3.

S3
-------
This is where is really becomes interesting as we have to make sure the data is written to S3 and S3 is replicated into another region __and__ to another account.

To do this you will have to configure two roles and two buckets. Both in different accounts each.

Let's start with creating a role in the source account which you will be assigning to the bucket in the source account. The source account is the account where the DynamoDB table is located.

You can either create a bucket through the AWS Console or use the following snippet:

```json
{
  "DynamoDbStreamToS3BucketRole": {
        "Properties": {
          "AssumeRolePolicyDocument": {
            "Statement": [
              {
                "Action": [
                  "sts:AssumeRole"
                ],
                "Effect": "Allow",
                "Principal": {
                  "Service": "s3.amazonaws.com"
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
                      "s3:GetObjectVersion",
                      "s3:GetObjectVersionTagging",
                      "s3:GetObjectVersionForReplication",
                      "s3:GetObjectVersionAcl"
                    ],
                    "Effect": "Allow",
                    "Resource": [
                      "arn:aws:s3:::source-account-dynamodb-backup/*"
                    ]
                  },
                  {
                    "Action": [
                      "s3:ListBucket",
                      "s3:GetReplicationConfiguration"
                    ],
                    "Effect": "Allow",
                    "Resource": [
                      "arn:aws:s3:::source-account-dynamodb-backup"
                    ]
                  },
                  {
                    "Action": [
                      "s3:ReplicateObject",
                      "s3:ReplicateDelete",
                      "s3:ReplicateTags",
                      "s3:GetObjectVersionTagging",
                      "s3:ObjectOwnerOverrideToBucketOwner"
                    ],
                    "Effect": "Allow",
                    "Resource": [
                      "arn:aws:s3:::destination-account-dynamodb-backup/*"
                    ]
                  }
                ],
                "Version": "2012-10-17"
              },
              "PolicyName": "DynamoDbStreamToS3BucketRole"
            }
          ]
        },
        "Type": "AWS::IAM::Role"
      }  
}
```

Next you need to create the source account bucket itself, keep in mind the bucket needs to be in the same region as where you did point your AWS Lambda function to (line 5). The bucket is configured with versioning (mandatory) and some lifecycle configuration policies to archive and cleanup these versioning. You can either copy these settings or define your own. Either way I would recommend you to set lifecycle policies as it will save you some extra bucks.

```json
{
  "DynamoDbStreamToS3Bucket": {
    "DeletionPolicy": "Retain",
    "Properties": {
      "AccessControl": "Private",
      "BucketName": "source-account-dynamodb-backup",
      "LifecycleConfiguration": {
        "Rules": [
          {
            "NoncurrentVersionExpirationInDays": 365,
            "NoncurrentVersionTransitions": [
              {
                "StorageClass": "GLACIER",
                "TransitionInDays": 30
              }
            ],
            "Status": "Enabled",
            "Transitions": [
              {
                "StorageClass": "GLACIER",
                "TransitionInDays": 30
              }
            ]
          }
        ]
      },
      "ReplicationConfiguration": {
        "Role": {
          "Fn::GetAtt": [
            "DynamoDbStreamToS3BucketRole",
            "Arn"
          ]
        },
        "Rules": [
          {
            "Destination": {
              "AccessControlTranslation": {
                "Owner": "Destination"
              },
              "Account": "<ACCOUNT B ID>",
              "Bucket": "arn:aws:s3:::destination-account-dynamodb-backup",
              "StorageClass": "STANDARD_IA"
            },
            "Prefix": "",
            "Status": "Enabled"
          }
        ]
      },
      "VersioningConfiguration": {
        "Status": "Enabled"
      }
    },
    "Type": "AWS::S3::Bucket"
  }  
}
```

And finally we have to create a bucket policy and a bucket in the destination account. To do so log into the destination account and create a bucket in a **different** region and make sure it also has versioning enabled. Optionally you can configure lifecycle configuration policies here too. The bucket policy is allowing the source account to replicate its content to the destination bucket.

First create the bucket:
```json
{
  "DynamoDbBackupBucket": {
    "DeletionPolicy": "Retain",
    "Properties": {
      "AccessControl": "Private",
      "BucketName": "destination-account-dynamodb-backup",
      "LifecycleConfiguration": {
        "Rules": [{
          "NoncurrentVersionExpirationInDays": 365,
          "NoncurrentVersionTransitions": [{
            "StorageClass": "GLACIER",
            "TransitionInDays": 30
          }],
          "Status": "Enabled",
          "Transitions": [{
            "StorageClass": "GLACIER",
            "TransitionInDays": 30
          }]
        }]
      },
      "VersioningConfiguration": {
        "Status": "Enabled"
      }
    },
    "Type": "AWS::S3::Bucket"
  }  
}
```

And lastly the bucket policy:
```json
{
  "DynamoDbBackupBucketPolicy": {
    "Properties": {
      "Bucket": {
        "Ref": "DynamoDbBackupBucket"
      },
      "PolicyDocument": {
        "Version": "2008-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {
              "AWS": "arn:aws:iam::<SOURCE ACCOUNT ID>:root"
            },
            "Action": [
              "s3:ReplicateObject",
              "s3:ReplicateDelete",
              "s3:ObjectOwnerOverrideToBucketOwner",
              "s3:List*"
            ],
            "Resource": [{
              "Fn::Join": [
                "", [
                  "arn:aws:s3:::",
                  {
                    "Ref": "DynamoDbBackupBucket"
                  },
                  "/*"
                ]
              ]
            }]
          },
          {
            "Effect": "Allow",
            "Principal": {
              "AWS": "arn:aws:iam::<SOURCE ACCOUNT ID>:root"
            },
            "Action": [
              "s3:GetBucketVersioning",
              "s3:PutBucketVersioning",
              "s3:List*"
            ],
            "Resource": [{
              "Fn::Join": [
                "", [
                  "arn:aws:s3:::",
                  {
                    "Ref": "DynamoDbBackupBucket"
                  }
                ]
              ]
            }]
          }
        ]
      }
    },
    "Type": "AWS::S3::BucketPolicy"
  }  
}
```

All set and done! Now we just have to verify everything is working. To do so change a record in DynamoDB and see if Lambda gets executed:
![dynamodb changed record](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-changing-record.png)
We can verify if Lambda has been executed accordingly at the Lambda Monitoring dashboard:
![lambda executed](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/lambda-executed.png)
If Lambda got executed correctly and without errors you should be able to see the `users` table as a folder in the source S3 backup bucket shortly.
![source backup folder](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-backup-folder.png)
The `users` folder will hold a separate folder for each DynamoDB record key ID that is changed. In this case `1274aad3-d77e-46cf-aa31-501f937bcd82` as this is the record we changed earlier:
![source backup folder content](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-backup-folder-content.png)
This folder will hold the final file with the complete JSON structure of our DynamoDB record:
![source backup folder file](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-backup-folder-file.png)
And finally, the file details page in S3 will show you the replication status. **COMPLETED** Success!
![source replication status](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-replication-status.png)

>In case you have a status **FAILURE** as replication status double check all your roles and policies. Make sure they are attached and have no typos.

Please note that Lambda will only write to S3 when event is posted to the DynamoDB Stream. Existing records in your DynamoDB table will not be written and replicated if they have not been changed.

In our next post we will write about how to restore from our replicated S3 bucket into DynamoDB. See you next time!
