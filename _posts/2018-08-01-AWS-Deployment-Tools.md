---
layout: post
title: AWS Deployment Tools Overview
banner: /assets/posts/2018-08-01-AWS-Deployment-Tools/3220b008-93a3-4c7a-a96a-d3ce15d0afcd.png
author: srknc
---

In Sentia MPC, we’re trying to use as much AWS services as possible - also for tooling. With this approach, lately we’ve decided to use AWS CodePipeline, together with other tools (CodeBuild, CodeCommit) for deployment process.

Before sharing the experience and the use-case, it’s better to explain tools in a nutshell.

## Definitions

**CodePipeline** is the orchestration / glue tool for your repositories, build and deployment steps. All these steps can use AWS solutions (CodeCommit, CodeBuild or CodeDeploy) but it’s also possible to use GitHub, Jenkins etc. 3rd tools for some steps.

**CodeCommit** is the equivalent of GitHub and Gitlab, authentication works with IAM. Although it has some important features like pull requests etc., most probably it’s handier just to sync it with GitHub or Gitlab.

**CodeBuild** is originally a building tool with YAML file task definition, but basically it can be used for any purposes as it’s possible to execute shell commands.

**CodeDeploy** is the deployment tool, it supports EC2 and Lambda (06.2018). Although it covers most important AWS services, It's not pretty straight forward to customise it or it doesn't make sense to you CodeDeploy if you're deploying using your own deployment scripts.

## Implementations

Our deployment workflow is simply something like this;

<img src="../../../assets/posts/2018-08-01-AWS-Deployment-Tools/3220b008-93a3-4c7a-a96a-d3ce15d0afcd2.png" alt="Deployment Model" style="width:400px;"/>



Starting from top to bottom, here is how we’ve implemented this workflow.

0- First step is deciding which account we’ll be using for CI/CD process. According to AWS (https://aws.amazon.com/blogs/devops/aws-building-a-secure-cross-account-continuous-delivery-pipeline/) If you’ve some requirements like, administrative isolation, limited visibility and discoverability of workloads, isolation in order to minimize blast radius, it’s necessary to use multiple accounts and put all CI/CD logic to a different account. Based on this, account strategy we’ve followed shown below;

<img src="../../../assets/posts/2018-08-01-AWS-Deployment-Tools/3220b008-93a3-4c7a-a96a-d3ce15d0afcd3.png" alt="Account Strategy" style="width:550px;"/>


1- Looking at deployment workflow again (image1), on step 1, first challenge is to get data for `Source Step`. Everything is very easy if you’re using CodeCommit but most probably it’s not the case. By default, CodeCommit supports GitHub as a source, but the problem is, if you’re using company/group profiles, when you want to give access to one of the repositories, you need to give access to all group, means the AWS account you’re using has access to all other projects under your company group. Otherwise you’ll need to create a read only account for each customers. This is only a problem if you’re using the web interface, with CloudFormation you can use `OAuthToken` at the source configuration step.
To solve this problem, we’ve used a cron to clone / pull data and and push to CodeCommit using some scripting;

git clone --mirror git@github.com:[companyname]/[repo].git [folder]
cd [folder]/
git remote add sync https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/[repo]
git push sync --mirror

So, after this workaround step, we use CodeCommit as source for deployment plans.

This is the `Source Step` of CodePipeline configuration;
```python
        "stages": [
            {
                "name": "Source",
                "actions": [
                    {
                        "inputArtifacts": [],
                        "name": "input-projectx-config",
                        "actionTypeId": {
                            "category": "Source",
                            "owner": "AWS",
                            "version": "1",
                            "provider": "CodeCommit"
                        },
                        "outputArtifacts": [
                            {
                                "name": "output-projectx-config"
                            }
                        ],
                        "configuration": {
                            "PollForSourceChanges": "false",
                            "BranchName": "develop",
                            "RepositoryName": "projectx-config"
                        },
                        "runOrder": 1
                    },
                    {
                        "inputArtifacts": [],
                        "name": "input-projectx-halloumi",
                        "actionTypeId": {
                            "category": "Source",
                            "owner": "AWS",
                            "version": "1",
                            "provider": "CodeCommit"
                        },
                        "outputArtifacts": [
                            {
                                "name": "output-projectx-halloumi"
                            }
                        ],
                        "configuration": {
                            "PollForSourceChanges": "true",
                            "BranchName": "feature/cloudformationchange",
                            "RepositoryName": "projectx-halloumi"
                        },
                        "runOrder": 1
                    }
                ]
            },
```

As it can be seen, both repositories trigger CodePipeline.


Next step at the workflow is combining these 2 repositories to be able to use it as source for CodeBuild process. The challenge here is, although one of two sources can trigger AWS CodePipeline, it doesn’t mean parallel source clone step brings codes from both repositories, it only brings the code it triggered CodePipeline.
To solve this problem, we’re using AWS CodeBuild and upload configuration repository to S3. So at the build step, CodeBuild uses S3 as source for config files and CodeCommit as source for CloudFormation code.

And this is the CodePipeline step for `Config Placement` step;
```python
            {
                "name": "Config-Placement",
                "actions": [
                    {
                        "inputArtifacts": [
                            {
                                "name": "output-projectx-config"
                            }
                        ],
                        "name": "projectx-config-placement",
                        "actionTypeId": {
                            "category": "Build",
                            "owner": "AWS",
                            "version": "1",
                            "provider": "CodeBuild"
                        },
                        "outputArtifacts": [
                            {
                                "name": "NA1"
                            }
                        ],
                        "configuration": {
                            "ProjectName": "projectx-halloumi-config-placement"
                        },
                        "runOrder": 1
                    }
                ]
            }
```

Here is the buildspec.yml file for `Config Placement` CodeBuild step.
```python
version: 0.2
env:
  variables:
    ENVIRONMENT: $ENVIRONMENT

phases:
  build:
    commands:
      - curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployments", "username":"codepipeline", "text":"['$ENVIRONMENT']:Config Placement Started"}' 'https://hooks.slack.com/services/xxxxxxxxxxxxxxxxxxxxxxx'
      - aws s3 cp . s3://projectx-deployment-artifacts/$ENVIRONMENT/config/ --recursive

artifacts:
  files:
    - '**/*'
```

So, whenever a change has been made at one of the repositories, this step uploads config repository code to S3 with environment name as prefix (folder), so next CloudFormation built step can download the code from here and combine it with the main repository..

Important to mention here, although it’s mandatory to use output-artifact to be able to put extra steps at CodePipeline, we’re not using those output artifacts as input for next step, we’ll be downloading the code from S3 at the build step.

## Build Step

3rd step on the workflow is CloudFormation build phase. Sentia has it’s own in-house CloudFormation generation engine, Halloumi. This step is basically responsible from getting the CloudFormation template engine code (from the output-artifact of Source step) and the configuration code from S3, executing Halloumi build command and pass artifacts (output of build command) to the next step.

Here is the .buildspec_build.yml CodeBuild step configuration file;
```python
version: 0.2
env:
  variables:
    foo: bar

phases:
  install:
    commands:
      - curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployments", "username":"CodePipeline", "text":"['$ENVIRONMENT']:Cloudformation Build Started"}' 'https://hooks.slack.com/services/xxxxxxxxxxxxxxxxxxxxxxx'
      - aws s3 cp s3://projectx-deployment-artifacts/ . --recursive
  build:
    commands:
      - bundle install
      - bundle exec rake run

  post_build:
    commands:
      - if [[ $CODEBUILD_BUILD_SUCCEEDING == 0 ]] ; then curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployments", "username":"codepipeline", "text":"['$ENVIRONMENT']:Cloudformation Build Failed:bangbang:"}' 'https://hooks.slack.com/services/xxxxxxxxxxxxxxxxxxxxxxx';  fi
      - if [[ $CODEBUILD_BUILD_SUCCEEDING == 1 ]] ; then curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployments", "username":"codepipeline", "text":"['$ENVIRONMENT']:Cloudformation Build Finished"}' 'https://hooks.slack.com/services/xxxxxxxxxxxxxxxxxxxxxxx';  fi
```
artifacts:
  files:
    - '**/*'


And CodePipeline configuration step for this build;
```python
            {
                "name": "Build",
                "actions": [
                    {
                        "inputArtifacts": [
                            {
                                "name": "output-projectx-halloumi"
                            }
                        ],
                        "name": "CodeBuild",
                        "actionTypeId": {
                            "category": "Build",
                            "owner": "AWS",
                            "version": "1",
                            "provider": "CodeBuild"
                        },
                        "outputArtifacts": [
                            {
                                "name": "output-projectx-halloumi-build"
                            }
                        ],
                        "configuration": {
                            "ProjectName": "projectx-halloumi-build"
                        },
                        "runOrder": 1
                    }
                ]
            },
```

Something the mention here, we’re keeping CodeBuild configuration files (buildspec.yml) (well, actually everything) at our code base. So, to be able to have more than one step, using buildspec_build.yml and buildspec_deploy.yml  files for different states.

On this step (at CodePipeline configuration) we mention output artifact as `build-output-artifact` and we’ll be using these files at the next step (CodeBuild - Deploy) as input artifact.


## Deploy Step
Last step at this CodePipeline configuration is the `deploy` step. AWS encourages customers to use CodeDeploy for this step. As we’ve explained before, CodeDeploy has limited features. For example if you need to execute a script after deployment, trigger an external system etc, it’s not possible. But it’s perfect if you’re deployment is only a CloudFormation deployment or a simple application.
In our case we’ve used CodeBuild to be able to deploy CloudFormation code and execute some extra steps.
Here is the buildspec_deploy.yml for CodeBuild step.
version: 0.2
```python
env:
  variables:
    ENVIRONMENT: $ENVIRONMENT
phases:
  install:
    commands:
      - curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployment", "username":"codepipeline", "text":"['$ENVIRONMENT']:Deploy Started"}' 'https://hooks.slack.com/services/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'
      - bundle install
  build:
    commands:
      - bundle exec rake run apply
  post_build:
    commands:
      - if [[ $CODEBUILD_BUILD_SUCCEEDING == 0 ]] ; then curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployment", "username":"codepipeline", "text":"['$ENVIRONMENT']:Deployment Failed:bangbang:"}' 'https://hooks.slack.com/services/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';  fi
      - if [[ $CODEBUILD_BUILD_SUCCEEDING == 1 ]] ; then curl -X POST --data-urlencode 'payload={"channel":"#projectx-deployment", "username":"codepipeline", "text":"['$ENVIRONMENT']:Deployment Finished:+1:"}' 'https://hooks.slack.com/services/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';  fi
artifacts:
  files:
    - '**/*'
```

And this is the CodePipeline `Build` step configuration;
```python
            {
                "name": "Deploy",
                "actions": [
                    {
                        "inputArtifacts": [
                            {
                                "name": "output-projectx-halloumi-build"
                            }
                        ],
                        "name": "projectx-halloumi-deploy",
                        "actionTypeId": {
                            "category": "Build",
                            "owner": "AWS",
                            "version": "1",
                            "provider": "CodeBuild"
                        },
                        "outputArtifacts": [
                            {
                                "name": "NA2"
                            }
                        ],
                        "configuration": {
                            "ProjectName": "acceptance-projectx-halloumi-deploy"
                        },
                        "runOrder": 1
                    }
                ]
            }
```

Deploy step consists of 3 main steps, install phase to install required packages, deploy phase and some curl commands to send slack messages.

This was the last step on Deployment process.

## Changing Deployment Branch

Another missing feature with CodePipeline is, not being able to easily change deployment branch for individual deployments. For preproduction and production, our customer deploys from master branch but they would like to deploy any branch to acceptance environment. To be able to achieve this, we’ve used Lambda and API Gateway.

<img src="../../../assets/posts/2018-08-01-AWS-Deployment-Tools/3220b008-93a3-4c7a-a96a-d3ce15d0afcd4.png" alt="Trigger and Update CodePipeline using API Gateway" style="width:800px;"/>


This API gateway has 2 methods, one is for changing the deployment branch, it triggers a lambda function which gets current CodePipeline configuration, changes branch definition and update CodePipeline again, using Boto3 library.
The other /trigger/ method only triggers CodePipeline execution without changing anything.

Here is the Lambda function code;

lambda_function.py
(codepipeline-trigger)

```python
import boto3
import json
import logging
import sys

logger = logging.getLogger()
logger.setLevel(logging.DEBUG)

def prepare_output(data,code):
    body={
      "data": data
    }
    out={
        "isBase64Encoded": False,
        "statusCode": int(code),
        "headers": {"Content-Type": "application/json","Access-Control-Allow-Origin":"*","Access-Control-Allow-Credentials":True},
        "body": ""
    }
    out['body'] = json.dumps(body)
    logger.debug(out)
    return out

def lambda_handler(event, context):
    logger.debug(event)
    request_json = json.loads(event['body'])
    key = request_json['key']
    pipeline_name = request_json['pipeline']

    pipeline_name = str(pipeline_name)
    key = str(key)

    if key != "XXXXXX":
        data = "key is not correct"
        return prepare_output(data,401)          

    # execute
    try:
        codepipeline = boto3.client('codepipeline')
        result = codepipeline.start_pipeline_execution(name=pipeline_name)
        logger.debug(str(result))
    except Exception as e:
        message = "could not update the CodePipeline" + str(e)
        return prepare_output(message,500)  

    return prepare_output("ok",200)
```


(codepipeline-change-branch)
lambda_function.py
```python
import boto3
import json
import logging
import sys

logger = logging.getLogger()
logger.setLevel(logging.DEBUG)

def prepare_output(data,code):
    body={
      "data": data
    }
    out={
        "isBase64Encoded": False,
        "statusCode": int(code),
        "headers": {"Content-Type": "application/json","Access-Control-Allow-Origin":"*","Access-Control-Allow-Credentials":True},
        "body": ""
    }
    out['body'] = json.dumps(body)
    logger.debug(out)
    return out

def lambda_handler(event, context):
    logger.debug(event)
    request_json = json.loads(event['body'])
    key = request_json['key']
    repository = request_json['repository']
    branch = request_json['branch']
    pipeline_name = 'acceptance-projectx'

    repository = str(repository)
    branch = str(branch)
    key = str(key)

    if key != "XXXXXXXXXXXX":
        data = "repositoy can be one of them: [projectx-config/projectx-halloumi]"
        return prepare_output(data,401)          

    if not repository in ["projectx-config","projectx-halloumi"]:
        data = "repositoy can be one of them: [projectx-config/projectx-halloumi]"
        return prepare_output(data,400)    

    data = "ok"
    logger.debug(data)

    # get CodePipeline
    codepipeline = boto3.client('codepipeline')
    try:
        current_pipeline = codepipeline.get_pipeline(
            name=pipeline_name
        )        
    except:
        return prepare_output("could not get current CodePipeline",500)            

    # find branch
    stages = current_pipeline['pipeline']['stages']
    # find "Source" stage
    stage_order=0
    for stage in stages:
        if stage.get("name") == "Source":
            break
        stage_order += 1
    action = (stage['actions'])

    # find the correct source by "RepositoryName"
    source_order = 0
    for action in stage['actions']:
        if action['configuration'].get("RepositoryName") == repository:
            break
        source_order += 1
    current_branch = action['configuration']['BranchName']


    print(current_pipeline)
 # update branch
    current_pipeline['pipeline']['stages'][stage_order]['actions'][source_order]['configuration']['BranchName'] = branch
    del current_pipeline['pipeline']['version']
    new_pipeline = current_pipeline['pipeline']

    # update CodePipeline
    try:
        update_output = codepipeline.update_pipeline(pipeline=new_pipeline)  
    except Exception as e:
        print(e)
        return prepare_output("could not update the CodePipeline",500)  
        print (update_output)

    ## release
    try:
        codepipeline.start_pipeline_execution(name=pipeline_name)
    except:
        print(e)
        return prepare_output("could not update the CodePipeline",500)  
        print (update_output)

    return prepare_output("ok",200)

````    


## Final Words


Advantages:   
* It's nice to be able to have workflow elements under one provider, AWS.
* There is no installation, servers, maintenance on all these services, it just works!
* It's cheap, price wise, really cheap comparing with having a CI/CD server and maintaining it.

Disadvantages:   
* It’s not possible to deploy a specific branch from web interface
* CodeDeploy support is very limited
*
