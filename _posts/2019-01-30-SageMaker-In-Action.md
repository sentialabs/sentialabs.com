---
layout: post
title: AWS SageMaker In Action
banner: /assets/posts/2019-01-30-SageMaker-In-Action/banner.png
author: mkianpour

---
Although Machine Learning is not new but recently it is quickly going forward thanks to public cloud providers such as AWS. By introducing [SageMaker](https://aws.amazon.com/sagemaker/) AWS is making Machine Learning more accessible and even more affordable to developers and data scientists. When combined with other AWS services such as [Glue](https://aws.amazon.com/glue/) which facilitates data engineering, AWS becomes a perfect place for practicing and deploying machine learning applications.

In this post we will not go over details of these services as AWS documentation is really informative and well organized. Instead we get our hands dirty by using them to see how they are useful for someone with little experience on Machine Learning. Most of the examples on AWS website are about image processing and I wanted to experience as much SageMaker tools and services as possible and from scratch. So, I defined a use-case of type text classification. We are a cloud company and it's important for us to know what's going on in public cloud! I want to build an analyzer who can say how much of the tweets posted on twitter about AWS are technical and how much marketing and commercial? The same with Azure so that no one says I'm biased! Anyway! Let's start!

The following is a general overview of the procedure.
![Overview](/assets/posts/2019-01-30-SageMaker-In-Action/sagemaker-overview.png)

## Step 0: Collect Data
We will use supervised learning techniques. In supervised learning, some datasets are required: Training, Validation, and Test datasets. We will talk about how to prepare data but the first step is to collect some relevant data. For this purpose we developed a simple Lambda function that collects tweets with a specific hashtag and stores in AWS S3.
You can find the code for the custom authorizer on [Github](https://github.com/mkianpour/twitter-machine-learning/blob/master/lambda/hashtag_collector.py)
```python

import boto3
import tweepy
import csv
import os
import datetime

def lambda_handler(event, context):
    ####input your credentials here
    consumer_key = os.environ['CONSUMER_AUTH_KEY']
    consumer_secret = os.environ['CONSUMER_AUTH_SECRET']
    access_token = os.environ['ACCESS_TOKEN']
    access_token_secret = os.environ['ACCESS_SECRET']

    hashtag = os.environ['HASHTAG']
    # hashtag = event["hashtag"]
    datalake_bucket = os.environ['DATALAKE_BUCKET']

    auth = tweepy.AppAuthHandler(consumer_key, consumer_secret)
    api = tweepy.API(auth,wait_on_rate_limit=True,wait_on_rate_limit_notify=True)
    # Open/Create a file to append data
    csvFile = open('/tmp/hashtag.csv', 'a')
    #Use csv Writer
    csvWriter = csv.writer(csvFile)

    hashtag_entry = f"#{hashtag}"
    s3 = boto3.resource('s3')

    yesterday = datetime.datetime.now() - datetime.timedelta(days=1)
    y_day = yesterday.strftime("%Y-%m-%d")
    print ("y_day: ", y_day, " yesterday: ", yesterday)

    for tweet in tweepy.Cursor(api.search,q=hashtag_entry,count=100,
                               since=y_day).items():
        csvWriter.writerow([hashtag,
                            tweet.user.name,
                            tweet.created_at,
                            tweet.text.encode('utf-8')])
    now = datetime.datetime.now()
    now_dir = now.strftime("%Y-%m-%d")
    s3.Bucket(datalake_bucket).upload_file("/tmp/hashtag.csv",
                                            f"twitter-data/hashtags/{hashtag}/{now_dir}/hashtag.csv")
```
We will use this Lambda function in several ways but here it's for collecting data. Each run will store all the tweets with specific hashtag in the past 24 hours. An example of the output is:

```
aws,vmiss,2019-01-04 17:52:54,b'The Truth About Virtual Machines In The Cloud https://t.co/j5Ay7x0nbb #azure #aws #gcp #cloud #ITPro'
aws,Lily's Social Media and Online Training,2019-01-04 17:50:45,b'JANUARY is for #AWS #CERTIFICATION\nFEATURED COURSES\nAWS Certified Solutions Architect - Professional 2019\n\nACE the\xe2\x80\xa6 https://t.co/jUM9Xy18xA'
aws,Jose Hidalgo Garcia,2019-01-04 17:50:39,b'RT @adhorn: My new post is out! Injecting Chaos to #AWS Lambda functions using Lambda Layers. Hope you enjoy :-) https://t.co/7ObEQKClg3 #se\xe2\x80\xa6'
aws,Angel Alejos,2019-01-04 17:50:21,b'RT @adhorn: My new post is out! Injecting Chaos to #AWS Lambda functions using Lambda Layers. Hope you enjoy :-) https://t.co/7ObEQKClg3 #se\xe2\x80\xa6'
aws,Gwen L Holland,2019-01-04 17:49:34,"b'Ace Info Solutions, Inc. is hiring a Cloud Automation Engineer in Bowie, MD #job #AWS #GovCloud https://t.co/HQPkcpt8tt'"
aws,josacar,2019-01-04 17:46:33,b'RT @adhorn: My new post is out! Injecting Chaos to AWS Lambda functions using Lambda Layers. Hope you enjoy :-) https://t.co/7ObEQKClg3 #se\xe2\x80\xa6'
aws,HYBRID TECHNOLOGY SYSTEMS,2019-01-04 17:46:02,b'How to install the Passbolt Team Password Manager on Ubuntu 18.04 https://t.co/6x7nMj0Dc1 via @hybrid_ts #AWS #Cloud https://t.co/2wSgD8kMSX'
aws,451 Research,2019-01-04 17:45:02,"b'#AWS may provide the tools, but expert #partner assistance will be needed to fully take advantage of those tools. F\xe2\x80\xa6 https://t.co/IFPTX3N0ye'"
aws,Intelligent Edge,2019-01-04 17:43:48,b'RT @IoTGN: Why deploying operational tech data to an OT/IT #cloud matters https://t.co/r3HmYtHa9F @MoxaInc\n #IIoT #EdgeComputing #IT #AWS #\xe2\x80\xa6'
aws,Bob Harris,2019-01-04 17:43:41,b'RT @richmerrett815: Very proud to have been involved with setting this new #AWS Alexa Skill Builder Certification. A great few days with so\xe2\x80\xa6'
aws,Sergio Cuéllar ☁️,2019-01-04 17:42:44,"b'RT @jeffbarr: Really cool - #AWS CLI Builder - https://t.co/WbagWpfJCx - ""Build your own AWS CLI commands..."" https://t.co/Kj4NM0x9pd'"
aws,AirwaySim,2019-01-04 17:41:47,b'Electrical blackout at Alicante - El Altet - flights cancelled #AWS #gameStatus #BW1'
aws,Whizlabs,2019-01-04 17:41:01,b'Best #Books for #AWS Certified Solutions #Architect #Exam https://t.co/OgYmzqMWNr via @whizlabs'
```

## Step 1: Prepare Training Data
As mentioned we use supervised learning techniques. So, we need some data to train the model (training dataset). From [AWS documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/how-it-works-mlconcepts.html):
>The type of data that you need depends on the business problem that you want the model to solve (the inferences that you want the model to generate). For example, suppose that you want to create a model to predict a number given an input image of a handwritten digit. To train such a model, you need example images of handwritten numbers.
Data scientists often spend a lot of time exploring and preprocessing, or "wrangling," example data before using it for model training. To preprocess data, you typically do the following:
Fetch the data— You might have in-house example data repositories, or you might use datasets that are publicly available. Typically, you pull the dataset or datasets into a single repository.

Although there are various open datasets but they are not usable in our scenario and we need to prepare training data ourselves. Basically we need to choose some data from what we collected in Step 0 and label those data. The amount of data should be big enough so that a good model can be created. This can be elaborated but again here we are mostly focused on how to use SageMaker rather than data science and data engineering. To label this piece of data we can use our own human resources but usually it's not easy to find people who can put time to label data. For example in this case they should go over thousands of tweets. AWS can help us to do this. Recently AWS launched Ground Truth:
![Ground Truth](/assets/posts/2019-01-30-SageMaker-In-Action/ground_truth.png)

We used Ground Truth by creating a new labeling job. It's almost straight forward. When creating a labeling job you specify the location of the file to be processed in an S3 bucket, you specify an S3 bucket as output location and of course IAM role with proper access to S3 buckets involved. Also you should specify the type of content that should be labeled.
![Labeling Job](/assets/posts/2019-01-30-SageMaker-In-Action/sagemaker-labeling.png)

In the next step it will ask you about the workforce you want to engage. Workforce can be public, your own employees or external verified people.
![Labeling Job](/assets/posts/2019-01-30-SageMaker-In-Action/sagemaker-labeling-2.png)

In this case we don't have any private information, so public workforce is ok. Actually AWS was providing this service as [AWS Mechanical Turk](https://www.mturk.com/) but it's integrated in AWS SageMaker page now. Also as you can see in the screenshot above, you should specify labels and give some instructions and examples to mechanical turks. Number of workers are adjustable as well. Labeling job can be costly, so before trying please check the pricing page for Ground Truth [here](https://aws.amazon.com/sagemaker/groundtruth/pricing/)!

Anyway, the output of labeling job would be an augmented manifest file. It's actually a `json` file specifying some details and a label per entry. An example of some entries in this manifest file is as follows:
```
{"source":"Chidambara .ML.,2019-01-04 17:34:02,b\u0027RT @ThingsExpo: CloudEXPO Silicon Valley Show Prospectus Published \\n\\nhttps://t.co/3QuZy0MwMl\\n\\n@Geek_King @TotalUptime #Cloud #IoT #IIoT #CI\\xe2\\x80\\xa6\u0027","hashtag-positivity-label-clone":1,"hashtag-positivity-label-clone-metadata":{"confidence":0.62,"job-name":"labeling-job/hashtag-positivity-label-clone","class-name":"Marketing","human-annotated":"yes","creation-date":"2019-01-08T18:53:17.309395","type":"groundtruth/text-classification"}}
{"source":"Ashot Nalbandyan,2019-01-04 17:33:00,b\u0027RT @dr_vitus_zato: How to Growth Stack Your Product \u003d\u0026gt; https://t.co/RMUFZKoNN0 #javascript #vuejs #code #php #angular #reactjs #redux #css\\xe2\\x80\\xa6\u0027","hashtag-positivity-label-clone":1,"hashtag-positivity-label-clone-metadata":{"confidence":0.92,"job-name":"labeling-job/hashtag-positivity-label-clone","class-name":"Marketing","human-annotated":"yes","creation-date":"2019-01-08T20:23:38.667523","type":"groundtruth/text-classification"}}
{"source":"InfoSec Industry,2019-01-04 17:32:32,b\u0027Kubernetes Security Issues (CVE-2018-18264 and kubectl proxy) https://t.co/Uxj1Q84f1N #AWS #infosec\u0027","hashtag-positivity-label-clone":0,"hashtag-positivity-label-clone-metadata":{"confidence":0.66,"job-name":"labeling-job/hashtag-positivity-label-clone","class-name":"Technical","human-annotated":"yes","creation-date":"2019-01-08T17:08:04.949567","type":"groundtruth/text-classification"}}
{"source":"Jamed,2019-01-04 17:30:42,b\u0027RT @NearShore_Tech: #Python opportunity in #Puebla and #Merida - #Django #AWS #Docker #Flask  Send your resume \\xe2\\x86\\x92 careers@nearshoretechnolog\\xe2\\x80\\xa6\u0027","hashtag-positivity-label-clone":1,"hashtag-positivity-label-clone-metadata":{"confidence":0.83,"job-name":"labeling-job/hashtag-positivity-label-clone","class-name":"Marketing","human-annotated":"yes","creation-date":"2019-01-08T19:37:00.318122","type":"groundtruth/text-classification"}}
{"source":"Lily\u0027s Social Media and Online Training,2019-01-04 17:30:36,b\u0027JANUARY is for #AWS #CERTIFICATION\\nFEATURED COURSES\\n\\nAWS Certified Developer Associate 2019\\nPass the AWS Certified\\xe2\\x80\\xa6 https://t.co/hAS1EbxeqA\u0027","hashtag-positivity-label-clone":0,"hashtag-positivity-label-clone-metadata":{"confidence":0.74,"job-name":"labeling-job/hashtag-positivity-label-clone","class-name":"Technical","human-annotated":"yes","creation-date":"2019-01-08T19:05:36.475779","type":"groundtruth/text-classification"}}
```

Now we have some labeled data which can be used by training jobs and ready to proceed with next step.

## Step 2: Prepare Training Data
In theory we are ready to train our model. Let's have a look at the following diagram which shows a typical ML workflow:
![ML workflow](/assets/posts/2019-01-30-SageMaker-In-Action/ml-concepts-10.png)

https://stackoverflow.com/questions/53962146/configure-training-job-using-ground-truth-and-blazingtext-in-amazon-sagemaker


If you have any questions, or if you would like to share your own Lambda runtimes ([ArnoldC](https://github.com/lhartikk/ArnoldC), [Brainfuck](https://esolangs.org/wiki/Brainfuck) or [L33t](https://en.wikipedia.org/wiki/Leet_(programming_language)), anyone?), reach out to me on [Twitter]( https://twitter.com/donkersgood).
