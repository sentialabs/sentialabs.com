---
layout: post
title: AWS SageMaker In Action
banner: /assets/posts/2019-02-19-SageMaker-In-Action/banner.png
author: mkianpour

---
In the [previous post](https://www.sentialabs.io/2019/01/30/SageMaker-In-Action.html) we collected and prepared training data to build the model. Now we continue with the building and deploying the model:

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

![Training Model](/assets/posts/2019-02-19-SageMaker-In-Action/model-in-console.png)

## Step 3: Deploying the Model

Now we have the model and we need to deploy it so that we can use it in our application (inference code). The following diagram shows how it works:

![Deploy Model](/assets/posts/2019-02-19-SageMaker-In-Action/sagemaker-architecture.png)

We can deploy this model again by some lines of code in Notebook or in Console. In Notebook, following the code above it's as easy as:

```
text_classifier = bt_model.deploy(initial_instance_count = 1,instance_type = 'ml.m4.xlarge')

```

In Console, the model can be deployed by clicking on the model and then specifying instance type for the machine which will host the inference code and interact with the application:

![Deploy Model in Console](/assets/posts/2019-02-19-SageMaker-In-Action/endpoint-creation.png)

## Step 4: Test, Verification and Fine tuning

At this point we can test and verify our model by giving some inputs and see how it works. This is how we tested this model:
```
sentences = ["AWS Insider,2019-01-04 17:12:07,b'#Verizon DevKit Targets #IoT Projects on #AWS #Cloud https://t.co/1ThmnTN3LR #Amazon'"]

# using the same nltk tokenizer that we used during data preparation for training
tokenized_sentences = [' '.join(nltk.word_tokenize(sent)) for sent in sentences]

payload = {"instances" : tokenized_sentences}

response = text_classifier.predict(json.dumps(payload))

predictions = json.loads(response)
print(json.dumps(predictions, indent=2))
```
remember that `text_classifier` was logical name of the endpoint. The result is:
```
[
  {
    "prob": [
      0.6029699444770813
    ],
    "label": [
      "__label__technical"
    ]
  }
]
```
I tried some more tweets and overall I was happy with the results.
We can continue more test but now we went over the whole cycle of SageMaker which was the purpose of this blog post. If we are not satisfied with the results, we can try to achieve better results by reviewing and refining training dataset or changing hyperparameters for the algorithm or even better algorithms but again these are data science topics and out of the scope of this post.

## Delete the Endpoint
Remember that ML instances are expensive. So, if you are done with the experiment and it's not in production! better to delete the endpoint before getting surprised after seeing the invoice!

As you see AWS SageMaker is taking a lot of heavy liftings from the shoulders of data engineers and data scientists and allows them to focus on their job. I also found it very useful for those who want to get a sense of Machine Learning and do some practices. I hope you have enjoyed this blog post and please let us know your feedback!
