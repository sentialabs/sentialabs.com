---
layout: post
title: Cross-account Metrics Forwarding
banner: /assets/posts/2018-04-13-ssm-secure-string/banner.png
author: sanjeev ranjan
---

Currently, AWS doesn't support sharing/forwarding of cross-account metrics for ecs cluster. This is just a matter of time though, as AWS will probably announce support at some point in the future, rendering this post obsolete.

The python script mentioned below can be used in the destination(/shared) account to fetch ecs metrics related to MemoryUtilization and CPUUtilization from the source accounts i.e. Production, Acceptance and Test. After some modification it can also be used for non-ecs metrics. This script needs to be triggered every minute.

```python
import json
import boto3
import time
import os

client = boto3.client('sts')
dest_cloudwatch_client = boto3.client('cloudwatch')

metric_names = [
    'CPUUtilization',
    'MemoryUtilization'
]

## function to fetch cloudwatch metrics
def fetch_json_cloudwatch(
    namespace,
    dimensions,
    cloudwatch_client,
    current_time
):
    metric_data_queries = []
    stats = [
        'Average',
        'Minimum',
        'Maximum',
        'Sum'
    ]
    for metric_name in metric_names:
        for stat in stats:
            metric_data_queries.append(
                {
                    'Id': 'request{}{}'.format(
                        metric_name,
                        stat
                    ),
                    'MetricStat': {
                        'Metric': {
                            'Namespace': namespace,
                            'MetricName': metric_name,
                            'Dimensions': dimensions
                        },
                        'Period': 60,
                        'Stat': stat
                    }
                }
            )

    res = cloudwatch_client.get_metric_data(
        MetricDataQueries=metric_data_queries,
        StartTime=(current_time - 120),
        EndTime=(current_time - 60)
    )
    return res

def lambda_handler(event, context):

    role_arns = [
        'PRODUCTION_CROSSACCOUNT_TRANSFER_ROLE_ARN',
        'ACCEPTANCE_CROSSACCOUNT_TRANSFER_ROLE_ARN',
        'TEST_CROSSACCOUNT_TRANSFER_ROLE_ARN'
    ]

    for role_arn in role_arns:
        response = client.assume_role(
            RoleArn=os.environ[role_arn],
            RoleSessionName='metricfetchsession'
        )
        session = boto3.Session(
            aws_access_key_id=response['Credentials']['AccessKeyId'],
            aws_secret_access_key=response['Credentials']['SecretAccessKey'],
            aws_session_token=response['Credentials']['SessionToken']
        )

        source_cloudwatch_client = session.client('cloudwatch')
        current_time = int(int(time.time())/60) * 60

        response = source_cloudwatch_client.list_metrics(
            Namespace='AWS/ECS',
            MetricName='CPUUtilization',
            Dimensions=[
                {
                    'Name': 'ClusterName'
                }
            ]
        )

        service_names = []
        for metric in response['Metrics']:
            service_name = ''
            cluster_name = ''
            for dimension in metric['Dimensions']:
                if dimension['Name'] == 'ClusterName':
                    cluster_name = dimension['Value']
                    break

            for dimension in metric['Dimensions']:
                if dimension['Name'] == 'ServiceName':
                    service_name = dimension['Value']
                    service_names.append(service_name)

                    metric_data_input = fetch_json_cloudwatch(
                        namespace='AWS/ECS',
                        dimensions=[
                            {
                                'Name': 'ClusterName',
                                'Value': cluster_name
                            },
                            {
                                'Name': 'ServiceName',
                                'Value': service_name
                            }
                        ],
                        cloudwatch_client=source_cloudwatch_client,
                        current_time=current_time
                    )
                    for metric_name in metric_names:
                        metric_data_output = {
                            'MetricName': metric_name,
                            'Dimensions': [
                                {
                                    'Name': 'ClusterName',
                                    'Value': cluster_name
                                },
                                {
                                    'Name': 'ServiceName',
                                    'Value': service_name
                                }
                            ],
                            'Values': [],
                            'StatisticValues': {
                                'SampleCount': 1,
                                'Sum': 0,
                                'Minimum': 0,
                                'Maximum': 0
                            },

                            'Timestamp': metric_data_input['MetricDataResults'][0]['Timestamps'][0],
                            'Unit': 'Percent'
                        }

                        for metric_name in metric_names:
                            if metric_name == 'CPUUtilization':
                                metric_data_cpu_output = metric_data_output
                            else:
                                metric_data_mem_output = metric_data_output

                        resources = metric_data_input['MetricDataResults']        

                        for resource in resources:
                            label = resource['Label']
                            value = resource['Values']
                            label_components = label.split(' ')
                            metric = label_components[0]
                            stat = label_components[1]

                            if metric == 'MemoryUtilization':
                                if stat == 'Average':
                                    metric_data_mem_output['Values'] = resource['Values']
                                else:
                                    metric_data_mem_output['StatisticValues'][stat] = resource['Values'][0]

                            if metric == 'CPUUtilization':
                                if stat == 'Average':
                                    metric_data_cpu_output['Values'] = resource['Values']
                                else:
                                    metric_data_cpu_output['StatisticValues'][stat] = resource['Values'][0]

                        metric_data_output = [
                            metric_data_mem_output,
                            metric_data_cpu_output
                        ]

                        dest_cloudwatch_client.put_metric_data(
                            Namespace='AWS/ECS',
                            MetricData=metric_data_output
                        )

```

This script takes as input role_arn:

Role arn -
It is arn of the role which is used to access CloudWatch metrics in source accounts.

How does this script work?

The script fetches cpu and memory data from different AWS accounts(source accounts). Using this data, It creates a CloudWatch metric in the namespace AWS/ECS with the same name as in the original account.


Functionality of code snippets of the lambda script has been described below.

Snippet 1 -
```python
def fetch_json_cloudwatch(
    namespace,
    dimensions,
    cloudwatch_client,
    current_time
):
    metric_data_queries = []
    stats = [
        'Average',
        'Minimum',
        'Maximum',
        'Sum'
    ]
    for metric_name in metric_names:
        for stat in stats:
            metric_data_queries.append(
                {
                    'Id': 'request{}{}'.format(
                        metric_name,
                        stat
                    ),
                    'MetricStat': {
                        'Metric': {
                            'Namespace': namespace,
                            'MetricName': metric_name,
                            'Dimensions': dimensions
                        },
                        'Period': 60,
                        'Stat': stat
                    }
                }
            )

    res = cloudwatch_client.get_metric_data(
        MetricDataQueries=metric_data_queries,
        StartTime=(current_time - 120),
        EndTime=(current_time - 60)
    )
    return res
```
This is the function which is used to fetch CloudWatch metrics. It fetches metrics from the time period between 2 minute(StartTime) before and 1 minute before(EndTime) the time of lambda trigger. The reason is that the most recent metrics(of the source account) can not be fetched as execution(/processing) of the code involve some time.
Different parameters of the function is described below.

a) Namespace - It is namespace of the CloudWatch metrics which is 'AWS/ECS' in this specific case.

b) Dimensions - The dimension(/identity) for the metrics. In this case, It constitutes name of the cluster(cluster_name) and different services(service_name).

c) MetricName - The name of the metric. In this case, It is CPUUtilization and MemoryUtilization.

d) current_time - Current time approximated to closest minute.

e) cloudwatch_client - Boto3 client for CloudWatch which is used to fetch metrics. It is the Cloudwatch client of the source account in this case.

f) stat - The CloudWatch statistic to return.

Snippet 2 -
```python
role_arns = [
    'PROD_CROSSACCOUNT_TRANSFER_ROLE_ARN',
    'QA_CROSSACCOUNT_TRANSFER_ROLE_ARN',
    'INT_CROSSACCOUNT_TRANSFER_ROLE_ARN'
]

for role_arn in role_arns:
    response = client.assume_role(
        RoleArn=os.environ[role_arn],
        RoleSessionName='metricfetchsession'
    )
    session = boto3.Session(
        aws_access_key_id=response['Credentials']['AccessKeyId'],
        aws_secret_access_key=response['Credentials']['SecretAccessKey'],
        aws_session_token=response['Credentials']['SessionToken']
    )

    source_cloudwatch_client = session.client('cloudwatch')
```

This piece of code facilitates read access to CloudWatch metrics in different accounts(source) from which metrics need to be forwarded. It is accomplished using different role arns for cross-account access. It loops over the role arns of the source accounts to create sessions.     

Snippet 3 -
```python
response = source_cloudwatch_client.list_metrics(
    Namespace='AWS/ECS',
    MetricName='CPUUtilization',
    Dimensions=[
        {
            'Name': 'ClusterName'
        }
    ]
)
```

This piece of code is used to list the metrics related to ECS cluster. Metrics is filtered against namespace, metric name and dimensions.

Snippet 4 -
```python
service_names = []
for metric in response['Metrics']:
    service_name = ''
    cluster_name = ''
    for dimension in metric['Dimensions']:
        if dimension['Name'] == 'ClusterName':
            cluster_name = dimension['Value']
            break

    for dimension in metric['Dimensions']:
        if dimension['Name'] == 'ServiceName':
            service_name = dimension['Value']
            service_names.append(service_name)
```

This piece of code is used to derive name of cluster and services.

Snippet 5 -
```python
metric_data_input = fetch_json_cloudwatch(
    namespace='AWS/ECS',
    dimensions=[
        {
            'Name': 'ClusterName',
            'Value': cluster_name
        },
        {
            'Name': 'ServiceName',
            'Value': service_name
        }
    ],
    cloudwatch_client=source_cloudwatch_client,
    current_time=current_time
)
```

This piece of code fetches metrics from source account for specified cluster and service name.  

Snippet 6 -
```python
metric_data_output = {
    'MetricName': metric_name,
    'Dimensions': [
        {
            'Name': 'ClusterName',
            'Value': cluster_name
        },
        {
            'Name': 'ServiceName',
            'Value': service_name
        }
    ],
    'Values': [],
    'StatisticValues': {
        'SampleCount': 1,
        'Sum': 0,
        'Minimum': 0,
        'Maximum': 0
    },

    'Timestamp': metric_data_input['MetricDataResults'][0]['Timestamps'][0],
    'Unit': 'Percent'
}
```

This piece of code create a dictionary in the format which can be used as input for the API(put_metric_data) which will be used to send metrics to destination account.

Snippet 7 -
```python
resources = metric_data_input['MetricDataResults']

for resource in resources:
    label = resource['Label']
    value = resource['Values']
    # timestamp = resource['Timestamps'][0]
    print(label)
    label_components = label.split(' ')
    metric = label_components[0]
    stat = label_components[1]

    if metric == 'MemoryUtilization':
        if stat == 'Average':
            metric_data_mem_output['Values'] = resource['Values']
        else:
            metric_data_mem_output['StatisticValues'][stat] = resource['Values'][0]

    if metric == 'CPUUtilization':
        if stat == 'Average':
            metric_data_cpu_output['Values'] = resource['Values']
        else:
            metric_data_cpu_output['StatisticValues'][stat] = resource['Values'][0]

metric_data_output = [
metric_data_mem_output,
metric_data_cpu_output
]
print(metric_data_output)
```

This piece of code retrieves the name of the metric and statistics from the fetched metrics. These names are used to filter fetched metrics so that the metric value of CPUUtilization and MemoryUtilization for different statistics is populated to corresponding dictionaries. Finally, both the dictionaries are used to create a list which can be as input for the put_metric_data API.

Snippet 8 -
```python
## publishing metrics in VAN IN:shared account
         dest_cloudwatch_client.put_metric_data(
             Namespace='AWS/ECS',
             MetricData=metric_data_output
         )
```         

This piece of code starts default session of the current(destination) account and then uses the put_metric_data API to put metric data in the CloudWatch metrics of the current(/destination) account.



NOTE - In order to not miss any metric data, this lambda needs to be triggered at an interval which is equal to or less than the difference between EndTime and StartTime of GetMetricData API.


Conclusion
----------
With this post I hope I have tried to explain how CloudWatch metrics can be forwarded to another account.
