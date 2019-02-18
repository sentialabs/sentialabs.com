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

## Step 2: Training the Model
In theory we are ready to train our model. Let's have a look at the following diagram which shows how training works:
![ML workflow](/assets/posts/2019-01-30-SageMaker-In-Action/sagemaker-architecture-training-2.png)
For a practical example using ready, labeled data you can see [AWS guide](https://docs.aws.amazon.com/sagemaker/latest/dg/ex1.html).

If you follow AWS guides, you see that the key for all Machine Learning operations is a Notebook instance. Those who have worked with [Jupyter Notebooks](https://jupyter.org/) already know how it works. Using GUI provided by Notebooks you can easily test your code or visualize the results, ... Required tools and libraries for developers are installed on the underlying machine. We can use Notebooks to train the model but recently AWS SageMaker added an option to facilitate modeling without the need to launch a Notebook and do some development. We wanted to give it a try:

![Training Job](/assets/posts/2019-01-30-SageMaker-In-Action/sagemaker-training-1.png)

As you see you will get an interface which you can specify the algorithm to be used and also Hyperparameters which are related to that specific algorithm. By using training jobs GUI, you don't have much visibility to what happens in the background and the idea is that the model is created for you. My personal experience was that this option is not helpful and doesn't work as expected and because of the lack of visibility you can't troubleshoot. Out of curiosity I tried this option to create a model but I had no success after more than 20 tries! I contacted AWS Support and they confirmed that they could regenerate the errors I was getting and they started investigation but after 1 month they couldn't figure it out:

>As an update, I am working with SageMaker Experts to figure out a workaround for the same. I will surely update you on the case the moment I will have something substantial to put forward to you.

So, I forgot this way of running training jobs. The better way to run training jobs would be to use Jupyter Notebooks to have more control over the operations. Using Notebooks in SageMaker we can fetch training data and proper training code (as containers), launch AWS ML instances and last but not least use training data and training code on ML instances to train the model. It saves the resulting model artifacts and other output in the S3 bucket we specified for that purpose.

For this practice we will use an algorithm provided by AWS SageMaker to train the model. The algorithm we chose is [BlazingText](https://docs.aws.amazon.com/sagemaker/latest/dg/blazingtext.html). Based on some investigation BlazingText algorithm can be used for text classification which fits our use case. The important thing is the input to the training code. Because we use available algorithms, the format of training data which is the input should follow the algorithm's specifications. As the guide for BlazingText reads:

>For supervised mode, the training/validation file should contain a training sentence per line along with the labels. Labels are words that are prefixed by the string '\__label__'. Here is an example of a training/validation file:
> "\__label__4".  linux ready for prime time , intel says , despite all the linux hype , the open-source movement has yet to make a huge splash in the desktop market . that may be about to change , thanks to chipmaking giant intel corp .
\__label__2  bowled by the slower one again , kolkata , november 14 the past caught up with sourav ganguly as the indian skippers return to international cricket was short lived .

### Step 2-1: Transforming Training Data

If you look at our training data, you see that the format is different. Even if we want to use manifested file as input, still some data engineering is required to transform the data. To transfer and clean the data we used another great service by AWS: [AWS Glue](https://aws.amazon.com/glue/)
We defined a Glue job to convert JSON file to a simple CSV as expected by our desired algorithm. It's pretty easy. As you see in the following picture we can easily map one field to a column in output or even remove unnecessary fields:

![Glue Job](/assets/posts/2019-01-30-SageMaker-In-Action/glue-job.png)

The result is the following which with a little tweak will be fit for BlazingText:
```
1,"Chidambara .ML.,2019-01-04 17:34:02,b'RT @ThingsExpo: CloudEXPO Silicon Valley Show Prospectus Published \n\nhttps://t.co/3QuZy0MwMl\n\n@Geek_King @TotalUptime #Cloud #IoT #IIoT #CI\xe2\x80\xa6'"
1,"Ashot Nalbandyan,2019-01-04 17:33:00,b'RT @dr_vitus_zato: How to Growth Stack Your Product =&gt; https://t.co/RMUFZKoNN0 #javascript #vuejs #code #php #angular #reactjs #redux #css\xe2\x80\xa6'"
0,"InfoSec Industry,2019-01-04 17:32:32,b'Kubernetes Security Issues (CVE-2018-18264 and kubectl proxy) https://t.co/Uxj1Q84f1N #AWS #infosec'"
```

We can now move to next phase which is actual training a model.

### Step 2-2: Training and Building the Model

To build the model we should have training code. Amazon SageMaker is quite flexible in using different algorithms. It also offers some ready to use algorithms. From [here](https://docs.aws.amazon.com/sagemaker/latest/dg/your-algorithms.html):

>Amazon SageMaker algorithms are packaged as Docker images. This gives you the flexibility to use almost any algorithm code with Amazon SageMaker, regardless of implementation language, dependent libraries, frameworks, and so on.

Here we use an available algorithm as mentioned: [BlazingText](https://docs.aws.amazon.com/sagemaker/latest/dg/blazingtext.html). We brought a Notebook up and use the algorithm to train the model. Bringing up a Jupyter Notebook is almost straight forward, as usual specify instance type and some AWS specific information such as VPC, encryption and most importantly IAM Role. When we have our Notebook running, we can run our code. You can see the whole notebook in [Github repository](https://github.com/mkianpour/twitter-machine-learning). The relevant code for training data is as follows:

```python
import sagemaker
from sagemaker import get_execution_role
import json
import boto3
import pandas as pd

sess = sagemaker.Session()

role = get_execution_role()
print(role) # This is the role that SageMaker would use to leverage AWS resources (S3, CloudWatch) on your behalf

bucket = "XYZ" # Replace with your own bucket name if needed
print(bucket)
data_key = 'twitter-data/hashtags/aws/2019-01-04/labeled.csv/hashtag-labeled-only-tweet.csv'
data_location = 's3://{}/{}'.format(bucket, data_key)

tweets = pd.read_csv(data_location)

tweets.to_csv('tweets-labeled.csv', sep=',', header=None, index=False)

prefix = 'twitter-data/marketing-model'

index_to_label = {}
#with open("dbpedia_csv/classes.txt") as f:
#    for i,label in enumerate(f.readlines()):
#        index_to_label[str(i+1)] = label.strip()
index_to_label = {
    "0": "technical",
    "1": "marketing"
}

def transform_instance(row):
    cur_row = []
    label = "__label__" + index_to_label[row[0]]  #Prefix the index-ed label with __label__
    cur_row.append(label)
    cur_row.extend(nltk.word_tokenize(row[1].lower()))
    return cur_row

    def preprocess(input_file, output_file, keep=1):
        all_rows = []
        with open(input_file, 'r') as csvinfile:
            csv_reader = csv.reader(csvinfile, delimiter=',')
            for row in csv_reader:
                all_rows.append(row)
        shuffle(all_rows)
        all_rows = all_rows[:int(keep*len(all_rows))]
        pool = Pool(processes=multiprocessing.cpu_count())
        transformed_rows = pool.map(transform_instance, all_rows)
        pool.close()
        pool.join()

        with open(output_file, 'w') as csvoutfile:
            csv_writer = csv.writer(csvoutfile, delimiter=' ', lineterminator='\n')
            csv_writer.writerows(transformed_rows)

preprocess('tweets-labeled.csv', 'tweets.train', keep=.8)
preprocess('tweets-labeled.csv', 'tweets.validation')

train_channel = prefix + '/train'
validation_channel = prefix + '/validation'

sess.upload_data(path='tweets.train', bucket=bucket, key_prefix=train_channel)
sess.upload_data(path='tweets.validation', bucket=bucket, key_prefix=validation_channel)

s3_train_data = 's3://{}/{}'.format(bucket, train_channel)
s3_validation_data = 's3://{}/{}'.format(bucket, validation_channel)

s3_output_location = 's3://{}/{}/output'.format(bucket, prefix)

#training
region_name = boto3.Session().region_name
container = sagemaker.amazon.amazon_estimator.get_image_uri(region_name, "blazingtext", "latest")
print('Using SageMaker BlazingText container: {} ({})'.format(container, region_name))

bt_model = sagemaker.estimator.Estimator(container,
                                         role,
                                         train_instance_count=1,
                                         train_instance_type='ml.c4.4xlarge',
                                         train_volume_size = 30,
                                         train_max_run = 360000,
                                         input_mode= 'File',
                                         output_path=s3_output_location,
                                         sagemaker_session=sess)

bt_model.set_hyperparameters(mode="supervised",
                           epochs=10,
                           min_count=2,
                           learning_rate=0.05,
                           vector_dim=10,
                           early_stopping=True,
                           patience=4,
                           min_epochs=5,
                           word_ngrams=2)

train_data = sagemaker.session.s3_input(s3_train_data, distribution='FullyReplicated',
                       content_type='text/plain', s3_data_type='S3Prefix')
validation_data = sagemaker.session.s3_input(s3_validation_data, distribution='FullyReplicated',
                            content_type='text/plain', s3_data_type='S3Prefix')
data_channels = {'train': train_data, 'validation': validation_data}
bt_model.fit(inputs=data_channels, logs=True)
```

If everything goes well we have our model at this point and it will be uploaded to output s3 bucket. The model can be seen in AWS Console too:

![Training Model](/assets/posts/2019-01-30-SageMaker-In-Action/model-in-console.png)

