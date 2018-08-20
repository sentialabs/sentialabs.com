---
layout: post
title: Building a Slack Bot with Serverless Framework
banner: /assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/banner.png
author: lvandonkersgoed

---

In this blog post we will explore how to build a Slack Bot utilizing Lambda, API Gateway and DynamoDB serverless technologies. We will define the environment using Serverless Framework.

# Serverless
The term "Serverless" can be a bit confusing. Within the AWS ecosystem, [serverless](https://aws.amazon.com/serverless/) is a term applied to technologies that:
- Can be used without any server management; all management and maintenance is done by AWS
- Have no idle capacity; the systems scale automatically to exactly meet demand
- Are automatically highly available; as a user of serverless technology there is literally nothing you need to do for HA
- Have a pay-per-use pricing model; you only pay for each function call, API call, database second or message you put or pull from a queue, but no base cost

Examples of AWS serverless technologies are: SQS, SNS, S3, Aurora Serverless, DynamoDB, Lambda, API Gateway, CloudWatch and Kinesis.

Keep in mind that "serverless" does not mean there are no servers involved.. that would be awkward. It just means that you do not need to *think* about servers when using these systems. It's entirely abstracted away.

Google Compute Cloud and Azure offer their own serverless solutions. More information about these technologies can be found at [cloud.google.com/serverless](https://cloud.google.com/serverless/) and [azure.microsoft.com/en-us/overview/serverless-computing](https://azure.microsoft.com/en-us/overview/serverless-computing/).

# Serverless Framework
The Serverless Framework is an entirely different, but related, beast. It's part of the Serverless Platform, hosted at [serverless.com](https://serverless.com/). The Serverless Platform and Serverless Framework are not affiliated with or part of any major cloud provider. 

The goal of the Serverless Platform is to, in their own words, "operationalize serverless development". Simply put, it's a wrapper around the serverless solutions offered by different cloud providers. Through automation and standardisation, it removes the heavy lifting of operating and versioning serverless applications.

# Slack Bots
Slack Bots are great for many reasons; they are fun and easy to build, are easy to interact with, and - with a bit of creativity - add value to your chat environment. Slack Bots are used for automation, interaction and streamlining processes. In our case, Slack Bots are a good way to become familiar with APIs and serverless technology.

The bot we will create today is a Karma Bot. If you upvote something or someone, it will add karma. If someone else downvotes it, the bot will subtract karma. 

From a technical perspective, there are multiple ways interact with Slack Bots. In this blog post we will focus on the [Events API](https://api.slack.com/events-api) to receive data *from* Slack and the [Web API](https://api.slack.com/web) to send data *to* Slack.

In combination with the AWS serverless technologies, this is the general architecture:
![Slack APIs](/assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/slack-apis.png)

# Getting started with Serverless Framework
First install the `serverless` application by following the [official instructions](https://serverless.com/framework/docs/getting-started/). Then create an empty folder for your project, and in it create a file called `serverless.yml`. Paste the following content.

```yaml
service: karmabot-tutorial

frameworkVersion: ">=1.1.0 <2.0.0"

provider:
  name: aws
  region: eu-west-1
  runtime: python3.6

functions:
  event_receive:
    handler: karmabot/event.receive
    memorySize: 128
    events:
      - http:
          path: event/receive
          method: post
          cors: true
```

Then create a folder called `karmabot`. In the folder, add a file called `event.py`. Your folder structure should now look like this:
```
karmabot-tutorial
|- karmabot
   |- event.py
|- serverless.yml
```

In a terminal window, move to your project's directory and execute `serverless package` or its shorthand variant `sls package`. Then list the directory's content with `ls -la`. You'll find that the serverless command has created a new directory `.serverless`, and in it two JSON files. Open `cloudformation-template-update-stack.json` and witness the power of the Serverless Framework.

With just a few lines of YAML and a single terminal command, Serverless has created about 350 lines of CloudFormation JSON containing 13 AWS resources, ranging from IAM Roles to API Gateway Deployments.

We could deploy this package to AWS, but it wouldn't do anything because we haven't added any code to the Lambda function yet. Let's do that now. Paste the following code to `event.py`:

```python
import json


def receive(event, context):
    data = json.loads(event['body'])
    print("Got data: {}".format(data))

    return {
        "statusCode": 200,
        "body": "ok"
    }
```

We now have an actual Lambda function, so it seems like a good idea to do a first deployment. If you haven't already, set up your AWS credentials as explained [here](https://serverless.com/framework/docs/providers/aws/guide/credentials/). Then run `sls deploy`. This will package your code and CloudFormation JSONs in the `.serverless` folder, create an S3 bucket in your account, upload the files, and initiate a CloudFormation stack deployment. You can see its progress in the CloudFormation web console.

When the deployment is done, check out the Lambda and API Gateway sections in the AWS web console. You'll find complete deployments of your code, including stages and routes. Browse to the `API Gateway -> your API -> stages -> dev` section and take note of the Invoke URL in the blue bar at the top of the page. You're going to need when registering your Bot at Slack.

![Invoke URL](/assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/api-stage.png)

# Reading Lambda logs
The Lambda function deployed to your AWS environment will output logs to CloudWatch. You can see follow the function's output in the AWS CloudWatch web console. However, the Serverless Framework also allows you to follow (or tail) the output in your terminal. To do this, open a new terminal screen and execute `sls logs --function event_receive -t`. There will be no output yet, but we will see something appear in the next step.

# Registering a Slack Bot
Go to [https://api.slack.com/apps](https://api.slack.com/apps) and register a new application. Then click **Event Subscriptions** in the left menu and toggle the **Enable Events** button.

This will display a number of options. The top one is the **Request URL**. This is the URL that Slack will send events to. We can select which events we want to receive later. For our bot, all events will need to be sent to our Lambda function. To achieve this, fill in the invoke URL from the previous step, suffixed with `/event/receive`. This path is the one we associated to our Lambda in `serverless.yml`. After you fill in the URL, you will receive an error:

![URL Verification](/assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/url-verification.png)

This makes sense, as Slack sends out a `challenge` and expects us to return the value it sends us. This is explained in the [Slack documentation](https://api.slack.com/events/url_verification). However, our Lambda function will always return a 200 and `ok` as its body, which is not what Slack wants. In the Lambda logs, which we started tailing in the previous step, you'll find output like this:

```
START RequestId: 968dfa80-a215-11e8-b868-c7304e011848 Version: $LATEST
Got data: {'token': 'HC3BhMRAzEnguQqzt20RKnWP', 'challenge': 'mg3msExXTM6v6pgAjQxjdX5h1QfwqB3RokOIdws8ibv9TKSNUBk6', 'type': 'url_verification'}
END RequestId: 968dfa80-a215-11e8-b868-c7304e011848
REPORT RequestId: 968dfa80-a215-11e8-b868-c7304e011848  Duration: 0.49 ms Billed Duration: 100 ms   Memory Size: 1024 MB  Max Memory Used: 23 MB  
```

If you're seeing this, your Lambda function successfully received data from Slack! Let's make sure it responds as it should. Open `event.py` and change its content to:

```python
import json


def receive(event, context):
    data = json.loads(event['body'])
    print("Got data: {}".format(data))
    return_body = "ok"

    if data["type"] == "url_verification":
        print("Received challenge")
        return_body = data["challenge"]

    return {
        "statusCode": 200,
        "body": return_body
    }
```

Next, run another `sls deploy` and wait for it to finish. Then go back to registering your Slack bot and click Retry. This should now work!

![Verified](/assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/verified.png)

In the terminal window running the `sls logs` command, you should also see the following output:
```
START RequestId: 0ed05834-a218-11e8-b90c-b1d98628fab8 Version: $LATEST
Got data: {'token': 'HC3BhMRAzEnguQqzt20RKnWP', 'challenge': 'Krb5zfJkXuDCbzlTnV3ufwAEsoUs6DHMINu1Q0WxIK4ywF1aBT8K', 'type': 'url_verification'}
Received challenge
END RequestId: 0ed05834-a218-11e8-b90c-b1d98628fab8
REPORT RequestId: 0ed05834-a218-11e8-b90c-b1d98628fab8  Duration: 0.52 ms Billed Duration: 100 ms   Memory Size: 1024 MB  Max Memory Used: 22 MB
```

Great success! Now let's make sure we can receive other events as well.

# Forwarding Slack Events to Lambda
Save any changes you've made on the Slack Events page and click **Bot Users** in the left menu. Then click **Add a Bot User** button. In the next screen change your bot's name if you like, then click **Save Changes**.

Next up, choose **Install App** in the left menu. Follow the steps, and you'll find a new bot in your workspace. Invite this bot to any channel you like. We'll configure the bot to forward any message posted in the channels it's in to our Lambda function.

Back in the Slack Bot console, go to the **Events Subscriptions** page again and enable the `message.channels` and `message.groups` events under **Subscribe to Workspace Events**. This will forward any message received in Slack channels to our Slack Bot.

![Subscriptions](/assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/subscriptions.png)

Click **Save Changes** and go back to your Slack Workspace. Send a message in any channel where your bot is a member, and you'll see the following in the log files:
```
START RequestId: a26eb2df-a2c9-11e8-990e-453b62f83234 Version: $LATEST
Got data: {'token': 'HC3BhMRAzEnguQqzt20RKnWP', 'team_id': 'T7SLPJM8U', 'api_app_id': 'AC9RD796V', 'event': {'type': 'message', 'user': 'U7SK63RLJ', 'text': 'hi <@UCADT4G3C>', 'client_msg_id': 'c23c7e11-fff6-43db-b27b-3cd4aaa6ba33', 'ts': '1534584753.000100', 'channel': 'C7VN60CSW', 'event_ts': '1534584753.000100', 'channel_type': 'channel'}, 'type': 'event_callback', 'event_id': 'EvCAUL59KL', 'event_time': 1534584753, 'authed_users': ['U7SK63RLJ']}
END RequestId: a26eb2df-a2c9-11e8-990e-453b62f83234
REPORT RequestId: a26eb2df-a2c9-11e8-990e-453b62f83234  Duration: 6.46 ms Billed Duration: 100 ms   Memory Size: 1024 MB  Max Memory Used: 23 MB  
```

Another great victory! Messages in channels are now sent to our Lambda script.

# Creating DynamoDB tables
The Serverless Framework allows us to easily set up DynamoDB tables and manage access to it. To do this we add a header `environment` to the `provider` section, and a new `resources` section:
```yaml
service: karmabot-tutorial

frameworkVersion: ">=1.1.0 <2.0.0"

provider:
  name: aws
  region: eu-west-1
  runtime: python3.6
  environment:
    KARMA_TABLE: ${self:service}-karma-${opt:stage, self:provider.stage}

functions:
  event_receive:
    handler: karmabot/event.receive
    memorySize: 128
    events:
      - http:
          path: event/receive
          method: post
          cors: true

resources:
  Resources:
    KarmaDynamoDbTable:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        AttributeDefinitions:
          -
            AttributeName: karma_id
            AttributeType: S
        KeySchema:
          -
            AttributeName: karma_id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:provider.environment.KARMA_TABLE}
```

Do a `sls deploy`, and you'll find a new DynamoDB table in your AWS environment. It's that easy! Next up: allowing our Lambda to read and write to the table. Update `serverless.yml` by adding the `iamRoleStatements` section, so the file looks like this:
```yaml
service: karmabot-tutorial

frameworkVersion: ">=1.1.0 <2.0.0"

provider:
  name: aws
  region: eu-west-1
  runtime: python3.6
  environment:
    KARMA_TABLE: ${self:service}-karma-${opt:stage, self:provider.stage}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: 
        - "arn:aws:dynamodb:${opt:region, self:provider.region}:*:table/${self:provider.environment.KARMA_TABLE}"

functions:
  event_receive:
    handler: karmabot/event.receive
    memorySize: 128
    events:
      - http:
          path: event/receive
          method: post
          cors: true

resources:
  Resources:
    KarmaDynamoDbTable:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        AttributeDefinitions:
          -
            AttributeName: karma_id
            AttributeType: S
        KeySchema:
          -
            AttributeName: karma_id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:provider.environment.KARMA_TABLE}
```

With this update, the Lambda function has all permission it needs. Now we will update the Lambda so it will actually read and write to the NoSQL database. Paste the following content to `event.py`:

```python
import json
import re
import decimal
import os
import boto3
import time
from boto3.dynamodb.conditions import Key

client = boto3.resource('dynamodb')
KARMA_TABLE = os.environ['KARMA_TABLE']
karma_table = client.Table(KARMA_TABLE)


def receive(event, context):
    data = json.loads(event['body'])
    print("Got data: {}".format(data))
    return_body = "ok"

    if data["type"] == "url_verification":
        print("Received challenge")
        return_body = data["challenge"]
    elif (
        data["type"] == "event_callback" and
        data["event"]["type"] == "message" and
        "subtype" not in data["event"]
    ):
        handle_message(data)

    return {
        "statusCode": 200,
        "body": return_body
    }


def handle_message(data):
    poster_user_id = data["event"]["user"]

    # handle all ++'s and --'s
    p = re.compile(r"(<?@.+?>?)(\+\+|--)")
    m = p.findall(data["event"]["text"])
    if m:
        for match in m:
            karma_word = match[0].strip()
            if karma_word == "<@{}>".format(poster_user_id):
                print("A user tried to change his own karma")
                warning = "Hey {}, you can't change your own karma!".format(
                    karma_word
                )
                send_message(data, warning)
                continue

            if not karma_exists(karma_word):
                create_karma(karma_word)

            if match[1] == "++":
                new_value = karma_plus(karma_word)
                reply = "Well done! {} now at {}".format(
                    karma_word, new_value
                )
            elif match[1] == "--":
                new_value = karma_minus(karma_word)
                reply = "Awww :( {} now at {}".format(
                    karma_word, new_value
                )
            send_message(data, reply)

    # handle all messages like `@test_word ==`
    p = re.compile(r"(<?@.+?>?)==")
    m = p.findall(data["event"]["text"])
    if m:
        for match in m:
            karma_word = match.strip()
            karma = get_karma_for_id(karma_word)
            if karma is None:
                karma = 0
            reply = "Karma for {}: {}".format(
                karma_word, karma
            )
            send_message(data, reply)


def get_karma_for_id(karma_word):
    result = karma_table.get_item(
        Key={
            'karma_id': karma_word.lower()
        }
    )
    if "Item" in result:
        return int(result["Item"]["karma"])
    else:
        return None


def karma_plus(karma_word):
    print("Adding karma for {}".format(karma_word))
    return karma_mod(karma_word, "+")


def karma_minus(karma_word):
    print("Subtracting karma for {}".format(karma_word))
    return karma_mod(karma_word, "-")


def karma_exists(karma_word):
    response = karma_table.query(
        KeyConditionExpression=Key('karma_id').eq(
            karma_word.lower()
        )
    )
    return response["Count"] > 0


def create_karma(karma_word):
    print("First karma for {}".format(karma_word))
    timestamp = int(time.time() * 1000)
    item = {
        "karma_id": karma_word.lower(),
        "karma": 0,
        "createdAt": timestamp
    }

    karma_table.put_item(Item=item)


def karma_mod(karma_word, sign):
    response = karma_table.update_item(
        Key={
            "karma_id": karma_word.lower()
        },
        UpdateExpression="set karma = karma {} :val".format(sign),
        ExpressionAttributeValues={
            ':val': decimal.Decimal(1)
        },
        ReturnValues="UPDATED_NEW"
    )
    return response["Attributes"]["karma"]


def send_message(data, text):
    print("Sending message to Slack: {}".format(text))
    pass
```

Don't worry if the code looks a bit complex, it's just an example of what you could do with a Slack Bot. In this case, the incoming message is processed in the `handle_message` function. In it, the message is scanned for `++`, `--` or `==`. The other functions provide the interaction with DynamoDB.

The only missing part is sending messages back to Slack. We will implement this in the next section.

# Sending messages to Slack
As described before, we will use the Slack Web API to send messages to Slack. To authenticate with the Slack API, we will need the **Bot User OAuth Access Token**, which you can find on the Slack Bot management page under the **OAuth & Permissions** section. Copy that value into the `environment` section of `serverless.yml`. The final version of `serverless.yml` should look like this:
```yaml
service: karmabot-tutorial

frameworkVersion: ">=1.1.0 <2.0.0"

provider:
  name: aws
  region: eu-west-1
  runtime: python3.6
  environment:
    KARMA_TABLE: ${self:service}-karma-${opt:stage, self:provider.stage}
    BOT_TOKEN: xoxb-264703633300-418469152114-wd8o6RBpDGHKfDpY3EROmrg6
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: 
        - "arn:aws:dynamodb:${opt:region, self:provider.region}:*:table/${self:provider.environment.KARMA_TABLE}"

functions:
  event_receive:
    handler: karmabot/event.receive
    memorySize: 128
    events:
      - http:
          path: event/receive
          method: post
          cors: true

resources:
  Resources:
    KarmaDynamoDbTable:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        AttributeDefinitions:
          -
            AttributeName: karma_id
            AttributeType: S
        KeySchema:
          -
            AttributeName: karma_id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:provider.environment.KARMA_TABLE}
```

Next, update `event.py` to its definitive version:
```python
import json
import re
import decimal
import os
import boto3
import urllib
import time
from boto3.dynamodb.conditions import Key

client = boto3.resource('dynamodb')
KARMA_TABLE = os.environ['KARMA_TABLE']
karma_table = client.Table(KARMA_TABLE)

BOT_TOKEN = os.environ['BOT_TOKEN']
SLACK_URL = "https://slack.com/api/chat.postMessage"


def receive(event, context):
    data = json.loads(event['body'])
    print("Got data: {}".format(data))
    return_body = "ok"

    if data["type"] == "url_verification":
        print("Received challenge")
        return_body = data["challenge"]
    elif (
        data["type"] == "event_callback" and
        data["event"]["type"] == "message" and
        "subtype" not in data["event"]
    ):
        handle_message(data)

    return {
        "statusCode": 200,
        "body": return_body
    }


def handle_message(data):
    poster_user_id = data["event"]["user"]

    # handle all ++'s and --'s
    p = re.compile(r"(<?@.+?>?)(\+\+|--)")
    m = p.findall(data["event"]["text"])
    if m:
        for match in m:
            karma_word = match[0].strip()
            if karma_word == "<@{}>".format(poster_user_id):
                print("A user tried to change his own karma")
                warning = "Hey {}, you can't change your own karma!".format(
                    karma_word
                )
                send_message(data, warning)
                continue

            if not karma_exists(karma_word):
                create_karma(karma_word)

            if match[1] == "++":
                new_value = karma_plus(karma_word)
                reply = "Well done! {} now at {}".format(
                    karma_word, new_value
                )
            elif match[1] == "--":
                new_value = karma_minus(karma_word)
                reply = "Awww :( {} now at {}".format(
                    karma_word, new_value
                )
            send_message(data, reply)

    # handle all messages like `@test_word ==`
    p = re.compile(r"(<?@.+?>?)==")
    m = p.findall(data["event"]["text"])
    if m:
        for match in m:
            karma_word = match.strip()
            karma = get_karma_for_id(karma_word)
            if karma is None:
                karma = 0
            reply = "Karma for {}: {}".format(
                karma_word, karma
            )
            send_message(data, reply)


def get_karma_for_id(karma_word):
    result = karma_table.get_item(
        Key={
            'karma_id': karma_word.lower()
        }
    )
    if "Item" in result:
        return int(result["Item"]["karma"])
    else:
        return None


def karma_plus(karma_word):
    print("Adding karma for {}".format(karma_word))
    return karma_mod(karma_word, "+")


def karma_minus(karma_word):
    print("Subtracting karma for {}".format(karma_word))
    return karma_mod(karma_word, "-")


def karma_exists(karma_word):
    response = karma_table.query(
        KeyConditionExpression=Key('karma_id').eq(
            karma_word.lower()
        )
    )
    return response["Count"] > 0


def create_karma(karma_word):
    print("First karma for {}".format(karma_word))
    timestamp = int(time.time() * 1000)
    item = {
        "karma_id": karma_word.lower(),
        "karma": 0,
        "createdAt": timestamp
    }

    karma_table.put_item(Item=item)


def karma_mod(karma_word, sign):
    response = karma_table.update_item(
        Key={
            "karma_id": karma_word.lower()
        },
        UpdateExpression="set karma = karma {} :val".format(sign),
        ExpressionAttributeValues={
            ':val': decimal.Decimal(1)
        },
        ReturnValues="UPDATED_NEW"
    )
    return response["Attributes"]["karma"]


def send_message(data, text):
    print("Sending message to Slack: {}".format(text))
    json_txt = json.dumps({
        "channel": data["event"]["channel"],
        "text": text
    }).encode('utf8')

    headers = {
        "Content-Type": "application/json",
        "Authorization": "Bearer {}".format(BOT_TOKEN)
    }

    req = urllib.request.Request(
        SLACK_URL,
        data=json_txt,
        headers=headers
    )
    urllib.request.urlopen(req)
```

In this version of the file, we've provided two new variables (`BOT_TOKEN` and `SLACK_URL`), and provided implementation to the `send_message` function. Deploy this code with `sls deploy`, and your bot should be completely functional!

You can verify if everything works in Slack. Just add some ++'s, --'s, and so on. Just don't forget to start anything, both things and users, with an @.

![Testing](/assets/posts/2018-08-16-Building-a-Slackbot-with-Serverless-Framework/testing.png)

# Cost
As said at the beginning of this post, with AWS serverless technologies you only pay for what you use. Let's break down what this bot would cost at significant use.

Let's assume a large workspace with a thousand users. These users on average send 50 messages per day, 5 days per week. The Karma Bot is present in all channels. One of the 50 messages per day is a karma interaction. The Karma Bot is hosted in the Ireland (eu-west-1) region.

This would mean a million messages per 4 weeks, and 20.000 of those are karma interactions. The first cost would be **API Gateway**. 1.000.000 messages would mean 1.000.000 API calls. API Gateway offers 1.000.000 free calls per month in the first year of use, so in that period there is no cost involved. After the first year, the price is $3.50 per million requests per month.

The second cost would be **Lambda** executions. Just like API Gateway, each message would mean 1 Lambda execution. Also like API Gateway, the first 1.000.000 Lambda executions per month are free. Unlike API Gateway however _this does not end after the first year_. So in our example, no cost for Lambda executions would be involved.

With Lambda you also pay for execution duration. This is calculated as GB-Seconds, or (reserved maximum memory the function can use * average execution time rounded up to nearest 100ms * number of executions). In our case, that's `128 MB * 100 ms * 1.000.000`, or `0,125 GB * 0.1 s * 1.000.000` = 12.500 GB-Seconds. The free tier offers 400,000 GB-Seconds per month for free, so this would also not invoke cost. This pricing available indefinitely, so our function would remain free after the first year.

The next cost segment is **DynamoDB**. DynamoDB pricing is split into three sections: storage in GBs, Read Capacity Units and Write Capacity Units. 

The records we store are 50 bytes each. Let's assume the worst and say that each of the 20.000 karma interactions create a new record. This would mean the database would be 20.000 * 50 bytes * 12 = 11.44 MB in size after *one year*. The first 25 GB are free, also indefinitely, so there is no cost involved for storage.

Each of the 20.000 interactions reads the DynamoDB table once. DynamoDB's free tier offers 129.600.000 reads of items up to 4 KB using eventually consistent reads (which we use). So that's free as well.

Each of the 20.000 interactions also writes to the DynamoDB table once. DynamoDB's free tier offers 64.800.000 writes of items up to 1 KB, so again.. there is no cost involved.

The final service we use is **CloudWatch**. It offers 5 GB of incoming logs per month for free, also indefinitely. Our logs won't even come close to that.

In conclusion, our service would be free in the first year, even under moderately heavy use. Past the first year, cost would be $3.50 per month.

# Conclusion
With this post, I hope I have displayed that with AWS serverless technology and the Serverless Framework, it's extremely easy and cheap to set up something cool like a Slack Bot. 50 lines of YAML, 160 lines of Python and about 2 hours of pointing and clicking is enough to get you there.

Of course, the Karma Bot we've built is not complete. I leave it to your imagination to extend it with whatever cool features you can think of. 

In an upcoming post I will also demonstrate how to uses AWS SQS and the SQS Lambda trigger to make the bot work asynchronously, and in another post I will show you how to use CodePipeline and CodeBuild to automatically deploy any changes made to your bot's Git master branch.
