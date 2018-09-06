---
layout: post
title: Data Continuity Service for DynamoDB
banner: /assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/banner.png
author: tvb
belongs_to: dcs.md
---

This is the first post of three where we are going to show how to build and configure a Data Continuity Service (DCS) for Amazon DynamoDB by using Amazon DynamoDB Streams, AWS Lambda, and the Amazon S3 services.

Overview
--------
![overview](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/solution-overview.png)


Definitions
--------
**Amazon DynamoDB** - Amazon DynamoDB is a fully managed proprietary NoSQL database service that supports key-value and document data structures.

**Amazon DynamoDB Streams** - Amazon DynamoDB Streams captures a time-ordered sequence of item-level modifications in any DynamoDB table, and stores this information in a log for up to 24 hours.

**AWS Lambda** - AWS Lambda is an event-driven, serverless computing platform. Run code without provisioning or managing servers.

**Amazon S3** - Amazon S3 provides object storage at scale.

**AWS IAM** - AWS Identity and Access Management (IAM) enables you to manage access to AWS services and resources securely.

Costs
--------
Make sure you have read the pricing pages for each service used so you are aware of any costs involved with this setup.


DynamoDB
-----------
For this post we are going to have a DynamoDB table called `users` which holds our company customers first name, last name, email address, and an important field. Just for the sake of example.

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

Note we stripped some of the other properties needed for easier reading purposes. The real magic is the property called `StreamSpecification` which holds the key "StreamViewType" and the value as a string __NEW_AND_OLD_IMAGES__. This is _crucial_.
>To read more about the CloudFormation properties available for your DynamoDB table see the [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html#cfn-dynamodb-table-streamspecification).

Once we have configured our DynamoDB Stream, the stream details on the DynamoDB overview page for the users table might look like the following:

![dynamodb-streams-enabled](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-streams-enabled.png)

We now have an Amazon Resource Name (ARN) available for us to use!

Lambda
--------
Ok so far it was easy going, right? We created a DynamoDB table and configured it to setup a DynamoDB Stream. Now lets move on to the part where the magic starts, AWS Lambda! AWS Lambda is really great as it is so powerful yet so flexible.

We are going to use Lambda to listen to the DynamoDB Stream and whenever there is an event posted onto the stream Lambda will read the data on the stream and write it to Amazon S3.

To configure Lambda we first need to create an IAM role so our Lambda script gets the permissions it needs when executing. Here is the CloudFormation snippet:

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
                "Resource": [
                  {
                    "Fn::Join": [
                      "",
                      [
                        "arn:aws:logs:",
                        { Ref: "AWS::Region" },
                        ":",
                        { Ref: "AWS::AccountId" },
                        ":*"
                      ]
                    ]
                  }
                ]
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
                "Resource": [
                  {
                    "Fn::Join": [
                      "",
                      [
                        "arn:aws:lambda:",
                        { Ref: "AWS::Region" },
                        ":",
                        { Ref: "AWS::AccountId" },
                        ":function:*"
                      ]
                    ]
                  }
                ]
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
                "Resource": [
                  {
                    "Fn::Join": [
                      "",
                      [
                        "arn:aws:dynamodb:",
                        { Ref: "AWS::Region" },
                        ":",
                        { Ref: "AWS::AccountId" },
                        ":table/users/stream/*"
                      ]
                    ]
                  }
                ]
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

A few things to note. First we are allowing Lambda to assume this role by specifying the `sts:AssumeRole` permissions under the "AssumeRolePolicyDocument" key.

Next we are defining the needed policies. We are allowing Lambda to write to CloudWatch Logs for logging purposes. We are also allowing Lambda to invoke the function or else it would not be able start at all. We are allowing Lambda to write to the source bucket and lastly we are allowing Lambda to access and read the DynamoDB Stream which is the hole point of this setup :smiley:

Once that is in place we can create our Lambda function and assign it to our newly created IAM role as an execution role. The Lambda function is written in NodeJS version 6.10.

![lambda-function](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/lambda-function.png)

Or use the CloudFormation snippet below:

```json
{
  "DynamoDbStreamToS3LambdaFunction": {
    "Properties": {
      "Code": {
        "ZipFile": "exports.backup = function (event, context, callback){ console.log('Hello DCS'); callback(null); }"
      },
      "Environment": {
        "Variables": {
          "BUCKET": "source-account-dynamodb-backup",
          "TABLE_NAME": "users",
          "REGION": { Ref: "AWS::Region" }
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

After the function is created we need to attach the DynamoDB Stream to the Lambda function so the function can ben triggered when there is an event posted:

![configure lambda 1](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/configure-lambda-trigger-1.png)

Resulting in:

![configure lambda 2](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/configure-lambda-trigger-2.png)

Or update your CloudFormation template with the content below. Make sure you replace `<STREAM_ID>` with the actual DynamoDB Stream Id shown on the overview page of your DynamoDB users table.
```json
{
  "DynamoDbStreamToS3LambdaEventSourceMapping": {
    "Properties": {
      "EventSourceArn": {
        "Fn::Join": [
          "",
          [
            "arn:aws:dynamodb:",
            { Ref: "AWS::Region" },
            ":",
            { Ref: "AWS::AccountId" },
            ":table/users/stream/<STREAM_ID>"
          ]
        ]
      },
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

const s3 = new aws.S3({ region: process.env.REGION });

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

It expects an environment variable named `TABLE_NAME` which holds the DynamoDB table name. In this case the table is called `users` so we are sending that with the function. It also expects another environment variable named `BUCKET` to define the source bucket name.

Actually, we have not really talked about buckets and S3 up to now so this would be a good moment to do so.

S3
-------
This is where it really becomes interesting as we have to make sure the data is written to S3 and S3 replicates the data into another region __and__ to another account.

To do this you will have to configure two roles and two buckets. We will put one role and bucket in the source account and the other role and bucket in the destination account.

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

Next you need to create the source account bucket itself. Keep in mind that the bucket needs to be in the same region as where you created your AWS Lambda function at. The bucket has to be configured with versioning enabled and you will probably want to configure lifecycle policies too to archive and cleanup versioned objects as it will save you some extra bucks. Also make sure you replace `<DESTINATION ACCOUNT ID>` with the AWS Account Id of the destination account.

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
              "Account": "<DESTINATION ACCOUNT ID>",
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

And finally we have to create a bucket policy and a bucket in the destination account. To do so, log into the destination account and create a bucket in a **different** region and make sure it also has versioning enabled. Optionally you can configure lifecycle configuration policies here too. The bucket policy is allowing the source account to replicate its content to the destination bucket. Make sure you replace `<SOURCE ACCOUNT ID>` with the AWS Account Id of the source account.

First create the bucket manually or use the CloudFormation snippet below:
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

And lastly the bucket policy in a CloudFormation template:
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

All set and done! Now we just have to verify everything is working now. To do so change a record in DynamoDB and see if Lambda gets executed:
![dynamodb changed record](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/dynamodb-changing-record.png)
We can verify if Lambda has been executed accordingly at the Lambda Monitoring dashboard:
![lambda executed](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/lambda-executed.png)
If Lambda was executed correctly and without errors you should be able to see the `users` table as a folder in the source S3 backup bucket shortly.
![source backup folder](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-backup-folder.png)
The `users` folder will hold a separate folder for each DynamoDB record key ID that is changed. In this case `1274aad3-d77e-46cf-aa31-501f937bcd82` as this is the record we changed in this example:
![source backup folder content](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-backup-folder-content.png)
This folder will hold the final file with the complete JSON structure of our DynamoDB record after it changed:
![source backup folder file](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-backup-folder-file.png)
And finally, the file details page in S3 will show you the replication status. **COMPLETED** Success!
![source replication status](/assets/posts/2018-09-05-Data-Continuity-Service-for-DynamoDB/source-replication-status.png)

>In case you have a status **FAILURE** as replication status double check all your roles and policies. Make sure they are attached and have no typos.

Please note that Lambda will only write to S3 when an event is posted to the DynamoDB Stream. An event will only be posted onto the stream when a record is changed. Existing records in your DynamoDB table will not be written and replicated if they have not been changed.

Conclusion
-------

In this post, we showed you how to use DynamoDB Streams together with Lambda to write changes made in your DynamoDB table to S3 and have S3 replicate this data into another region and to another account. This way you can be sure you are having backups somewhere save if disaster really strikes.

If you have any questions or remarks about this blog please reach out to us through our [Slack channel](http://www.sentialabs.io/contact.html).

In our next post we will show how to restore from our replicated S3 bucket back into DynamoDB. See you next time!
