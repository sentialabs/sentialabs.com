---
layout: post
title: Integrating EKS with other AWS Services
banner: /assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/banner.png
author: lvandonkersgoed

---
EKS offers developers an easy way to run Kubernetes workloads at AWS. But what if you need to integrate your EKS based app with other services like CloudFront, API Gateway or Web Application Firewall?

For a recent project we needed to deploy a number of environments with EKS clusters. Each of these environments had different requirements regarding the supporting services; some needed API Gateway, some needed WAF, others needed CloudFront, and many required a combination of these services. We've worked to create a "one size fits all" deployment for EKS - a configuration that allows for integration with all of these services. You can find our results below.

Re:Invent 2018 is being held in Las Vegas next week. We expect that AWS will announce more supported regions for EKS, leading to more EKS deployments in the near future. We hope this blog post will help you set up the integrations and complex configurations for all your use cases.

The challenges we will be discussing fall into two categories:
- Reliably integrating HTTP frontend services (CloudFront, API Gateway and WAF) with EKS
- Using Infrastructure as Code (CloudFormation) to deploy resources that depend on EKS managed resources

## About EKS
Elastic Container Service for Kubernetes (somehow abbreviated to EKS) is Amazon's implementation of a managed Kubernetes service. In a nutshell, Amazon provides the master nodes, you provide the worker nodes, you do some configuration, and voilà: you have a highly available, scalable, and relatively cheap Kubernetes cluster. 

Some EKS specific features:
- The worker nodes for EKS are provided by EC2 instances in an autoscaling group. Fargate support for EKS is expected to be released soon.
- You can assign IAM roles to the worker nodes to allow interaction with other AWS services like CloudWatch and SQS.
- You can use a magically altered version of kubectl to access Kubernetes with IAM credentials.
- With [Kubernetes Autoscaler](https://github.com/kubernetes/autoscaler) EC2 nodes can be added to and removed from the cluster dynamically, based on the resources demanded by the Pods.
- Using annotation, you can tell Kubernetes what kind of load balancers to deploy in the AWS environment - either internal or internet-facing, and either Classic Load Balancer or Network Load Balancer.

Does this sound awesome? Well, that's because it is. But it's not all unicorns and puppies in EKS land. Some integrations are a real hassle to set up. This post aims to help you achieve these more complex setups.

## First challenge: many services, many load balancers
Within Kubernetes you can define Services. From the official Kubernetes documentation: 
> A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service.

Simply put, a Service is a collection of Pods. The Service is reachable at a consistent endpoint (either internal or external facing), no matter what happens to the underlying Pods.

In your Service definition, you can tell Kubernetes to deploy a load balancer for your service. In EKS, this will result in an actual AWS Elastic Load Balancer (ELB) in your account. See this example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-python
  labels:
    run: hello-python
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
    protocol: TCP
    name: http
  selector:
    run: hello-python
```

This would deploy a classic, internet facing load balancer.

But what would happen if you defined five services? You would get five load balancers. And this is a problem when you want to deploy a frontend technology like the Web Application Firewall (WAF): it can only handle a single load balancer. API Gateway can handle multiple VPC Links, CloudFront can have multiple origins, but updating API Gateway or CloudFront every time you have a new service is very inconvenient. Also, when you would like to terminate a Service, Kubernetes would be unable to clean up the load balancer because it would be in use by the frontend service.

The solution: use an [Ingress](https://github.com/kubernetes/ingress-nginx) service in Kubernetes. An Ingress is like a router or gateway for your cluster: you can define which traffic goes to which internal service, based on hostnames or paths. The Ingress service is the only publicly accessible component of your Kubernetes cluster. The backend services are only reachable from within the Kubernetes cluster (and as such, have no AWS load balancers of their own).

![Ingress](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/ingress.png)

There are many benefits to using Ingress, but for the scope of this blog post the best part is that using Ingress results in only one ELB, which we can reliably use for our frontend services.

## Second challenge: classic load balancer, application load balancer or network load balancer?
In the previous chapter we've described how to make sure you have only one load balancer in front of your Kubernetes cluster. If you add no additional configuration to your Ingress service definition, it will create a Classic Load Balancer (CLB) for you. However, CLBs are sort of deprecated. Additionally, CLBs don't play nice with Web Application Firewall (WAF), which can only be attached to an Application Load Balancer (ALB). API Gateway doesn't like CLBs either; if you want to put API Gateway in front of a private service, it can only route traffic to Network Load Balancers (NLBs), using VPC Links. 

So WAF requires an ALB, API Gateway wants an NLB, and CloudFront doesn't care. But Kubernetes itself further limits your choice in load balancers; at the moment of writing you can only reliably configure a Kubernetes service to deploy a CLB or an NLB. There is a [project](https://github.com/kubernetes-sigs/aws-alb-ingress-controller) that enables the deployment of ALBs, but it is maintained by a third party. An additional reason we don't want to let the ALB be created by Kubernetes is because we want to configure its SSL certificates, security groups and WAF through CloudFormation. We will dive into this at the end of the blog post.

This pretty much leaves us with only one option: using the NLB. Fortunately this is very new and powerful technology, which will allow many versatile solutions. More about that later in this post (hint: we can still use WAF).

To tell Kubernetes to deploy an NLB instead of a CLB, we use annotation. Annotations are key-value pairs you can add to pretty much any object in Kubernetes (Pod, Service, Namespace, and so on). These annotations can be read by internal and external systems, which can use them as a trigger or modifier on certain actions. In this case we use annotations to "tell" Ingress to deploy a Network Load Balancer instead of a Classic Load Balancer. The annotation that achieves this is `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"`. An example:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
```

Deploy the Ingress Service using the specification above, and the Kubernetes cluster will deploy a Network Load Balancer for you.

## Third challenge: defining in which subnets your load balancers will be placed
In the world of AWS Elastic Load Balancers, there are two forms of connectivity: internal (or private) and internet-facing (or public). As the names suggest, the first one can only be reached from the internal network (e.g. EC2 instances, connections over VPN and Lambdas in your VPC). Public ELBs can be reached over the internet and are very useful for public web services.

From a security and networking perspective, you need different infrastructure for private and public load balancers; private load balancers should be placed in subnets that do not have a route to an Internet Gateway (IGW), also called private subnets. Public load balancers can only function in subnets that *do* have a route to an IGW.

Within Kubernetes, you can specify whether a load balancer should be public or private through annotation. If you add the annotation `service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0` to a service, it will create an internal load balancer. To show this in context:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
```

Deploying the Service definition above will result in an internal NLB. And while this flexibility is very nice, it poses a problem: how should you "tell" Kubernetes which subnets to use for the load balancers? After all, it shouldn't deploy a public load balancer in a private subnet.

The solution is to use tags with your subnets. These tags identify which subnets are private and which are public, allowing Kubernetes to choose the correct ones for its load balancers. The tags to use for internal subnets are:
```
kubernetes.io/cluster/<Your EKS cluster ID>: shared
kubernetes.io/role/internal-elb: 1
```
After replacing `<Your EKS cluster ID>` with your cluster ID, it should look like this in the AWS console:
![Subnet Tags](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/subnet_tags.png)

For public subnets the keys are almost the same, but without the `internal-` prefix:
```
kubernetes.io/cluster/<Your EKS cluster ID>: shared
kubernetes.io/role/elb: 1
```

When writing this article, documentation about these tags was very sparse. In fact we found the correct tags in the Kubernetes [source code](https://github.com/kubernetes/kubernetes/blob/master/pkg/cloudprovider/providers/aws/aws.go). If you're ever searching for undocumented functionality, we strongly advise to read through that file.

To be flexible about deploying public or private load balancers for EKS, the easiest solution is to deploy both public and private subnets. That way, whatever you do from within Kubernetes, the cluster will have a place for its load balancers. If you're deploying in a region with three availability zones, this would mean creating six subnets.

A diagram displaying multiple subnets with an internal NLB deployed:
![Subnets](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/subnets.png)

## Fourth challenge: getting WAF to work with an NLB
Kubernetes is great for web services; you can easily create complex architectures for micro services, each scaling independently. You can add worker processes, scheduled processes and much more, all the while improving your development pace and security posture. But not all problems should be solved in Kubernetes. In general, specialistic services are better at solving specialistic tasks than generalistic solutions like Kubernetes are. For example, if you're hosting in AWS, you want to use RDS for relational databases, ElastiCache for in-memory caching and S3 for storage. You *could* provide all of these services within the cluster, but you shouldn't want to.

Another example of a managed service doing its task a lot better than you probably ever could achieve yourself is AWS Web Application Firewall (WAF). It inspects all incoming traffic and can very quickly block attempts at SQL injection, cross site scripting, brute force attacks and much more.

As mentioned above, there is a caveat: WAF requires a service that handles HTTPS offloading to integrate with - WAF can't do that by itself. The services that WAF is compatible with are API Gateway, CloudFront and Application Load Balancer (ALB). If your architecture includes one of the first two options, please use them to host the WAF. However, if you don't need API Gateway or CloudFront, it's quite a waste to just deploy them as a "glue" component for WAF. Especially in high traffic environments (which is generally where you want to use a web application firewall), this can get real expensive real fast.

Additionally, for the reasons described in the second challenge, we don't want Kubernetes to deploy an ALB for us. So let's see how we can deploy an Application Load Balancer in front of our EKS cluster.

Start by making sure you're using Ingress (as described in the first challenge), with a Network Load Balancer (as described in the second challenge), with the internal scheme in internal subnets (as described in the third challenge). This guarantees that all traffic to the EKS cluster goes through the NLB and the cluster can only be reached from within the VPC.

The ALB will be public (and thus reachable over the internet), but its targets should point to the Network Load Balancer. The ALB has two target types: `instance` or `ip`. We're not connecting to EC2 instances, so we'll have to find a solution with IP addresses.

As you may know, all types of AWS load balancers are reachable at a DNS name, and you should never store or connect directly to the underlying IP addresses. The reason is that when ELBs scale, the actual servers doing the load balancing might be replaced by more powerful instances with different IP addresses. The DNS name will stay the same, however.

Network Load Balancers do support static IP addresses through Elastic IPs, but only for public IP addresses. We require the private addresses to be static, which is not an available option.

**VPC Interface Endpoints to the rescue**

The VPC Endpoints technology allows developers to publish application services and let other environments connect to them in a very secure and highly performant manner. From the AWS [documentation](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-service.html):
> You can create your own application in your VPC and configure it as an AWS PrivateLink-powered service (referred to as an endpoint service). Other AWS principals can create a connection from their VPC to your endpoint service using an interface VPC endpoint. You are the service provider, and the AWS principals that create connections to your service are service consumers.

![VPC Endpoints](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/vpc_endpoint.png)

An example use case would be a central authorization service that is being used by hundreds of external applications in separate VPCs. The authorization service would be hosted on a fleet of EC2 instances reachable through a Network Load Balancer. All the consumers could configure VPC peerings and routes, but this would cost a lot of time and there would most likely be issues with overlapping CIDRs. With VPC Endpoint Services, the internal Network Load Balancer for the authorization service would be published as a VPC Endpoint Service. The consumers can subscribe to this service, which allows them to directly connect to the NLB over the AWS backbone.

The [PrivateLink](https://aws.amazon.com/privatelink/) technology powering this system requires the consumer to specify a number of subnets in their VPC, one per availability zone. In each of these AZs, an Elastic Network Interface (ENI) is deployed. The ENIs are reachable on a custom DNS name. Sending traffic to this DNS name (and thus the network interfaces) routes the traffic directly to the Network Load Balancer in the other account. But here's the kicker: *these NICs have static IP addresses*.

With this knowledge in hand, we can use AWS VPC Endpoint Services to make the NLB reachable at static addresses, which can be used as IP targets by the ALB. In a diagram:

![VPC Endpoints with ALB](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/vpc_endpoint_alb.png)

This solution does incur some cost: AWS [charges](https://aws.amazon.com/privatelink/pricing/) about $0.01 per VPC Endpoint per AZ (around $22 per 30 days for three AZs) and $0.01 per gigabyte processed (about $100 per 10TB). If you're processing very large amounts of data, this solution might not be the best fit.

## Fifth challenge: deploying API Gateway in front of EKS
API Gateway can be used as a frontend for private resources in your VPC. With [API Gateway Private Integrations](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-private-integration.html) your private resources are protected from access over the internet, so you have the guarantee that all access to your cluster goes through API Gateway. This allows you to enforce authorization, the use of API keys and rate limiting. With CloudFront, this is a lot harder to achieve - CloudFront can only use public endpoints as its endpoints. We'll talk more about CloudFront in the next chapter.

API Gateway is a service that lives outside your VPC; it is completely hosted and maintained in an AWS owned VPC, and its underlying infrastructure is invisible to you. API Gateway uses AWS PrivateLink technology to connect to your private resources. As described in the previous chapter, PrivateLink publishes services based on Network Load Balancers. As such, API Gateway can only connect to your private resources through an internal NLB. In the API Gateway console, you can create a VPC Link pointing to this NLB. The VPC Link can be used in your API Gateway definitions. 

This is all a lot easier to digest with a diagram:
![API Gateway](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/api_gateway.png)

## Sixth challenge: deploying CloudFront with EKS as its origin
This is hardly a problem, compared with the previous challenges. CloudFront only requires an existing public DNS name to configure as its origin. So for CloudFront it doesn't even matter if you chose a CLB or an NLB for Ingress; you can just point CloudFront to the load balancer's DNS endpoint.

![CloudFront](/assets/posts/2018-10-21-Integrating-EKS-with-other-AWS-services/cloudfront.png)

However, keep in mind that the load balancer needs to be accessible over the internet. By definition, this means CloudFront (and optionally a WAF deployed with CloudFront) can be bypassed if someone knows the address of the load balancer. A common solution to this problem is to apply security groups at the load balancer, which only allow traffic from known CloudFront CIDRs. But here's the catch: NLBs don't support security groups, because they are built to be transparent and apply no modification or shaping to traffic whatsoever. So in this scenario you can either revert to using CLBs or apply the ALB-in-front-of-NLB solution described at the fourth challenge.

## Seventh challenge: integrating EKS managed resources with CloudFormation
At Sentia we deploy everything through Infrastructure as Code (IaC) using CloudFormation (CFN). CFN allows you to specify most AWS services and resources through JSON or YAML templates. Deploying the EKS infrastructure as described in this blogpost with CFN poses a big challenge: EKS deploys load balancers in your AWS environment, but CFN needs to be "aware" of this load balancer so other services can integrate with it.

Our solution is taking a staggered deployment approach:
1. Deploy an empty EKS Cluster with CFN
2. Deploy a Lambda function that runs every minute (more about this later)
3. Connect to EKS and deploy an Ingress with a Network Load Balancer as described in this blog post
4. The lambda detects the load balancer created by EKS and stores its ARN in the Parameter Store
5. Redeploy the CFN stacks, this time with API Gateway or ALB, using the ARN in the Parameter Store

The "magic" component in this setup is without a doubt the Lambda function. Here is the full source code:

```python
import os
import boto3
import datetime
import logging

PARAMETER_KEY = os.environ["NLB_ARN_PARAMETER_KEY"]
HOSTED_ZONE_ID = os.environ["HOSTED_ZONE_ID"]
DOMAIN_NAME = os.environ["DOMAIN_NAME"]

ssm_client = boto3.client("ssm")
elb_client = boto3.client("elbv2")
r53_client = boto3.client("route53")

def handler(event, context):
  print(event)
  find_eks_nlb()


def find_eks_nlb():
  """
  find_eks_nlb() checks all load balancers for an internal NLB.
  If it finds it, it checks its tags to verify it's the Ingress NLB.
  If the NLB is the Ingress NLB, it will store its ARN in the SSM
  Parameter Store.
  """
  eks_nlb_found = False
  eks_nlb_arn = None

  lbs = elb_client.describe_load_balancers()
  ingress_lbs = filter_ingress_tags(lbs["LoadBalancers"])
  for lb in ingress_lbs:
    if lb["Scheme"] == "internal":
      arn = lb["LoadBalancerArn"]
      print("Found internal NLB: {}".format(arn))
      eks_nlb_found = True
      eks_nlb_arn = arn
    elif lb["Scheme"] == "internet-facing":
      arn = lb["LoadBalancerArn"]
      print("Found public NLB: {}".format(arn))

    update_route_53(lb)

  if eks_nlb_found:
    store_arn_in_parameter_store(eks_nlb_arn)
  else:
    clear_parameter_store()


def update_route_53(lb):
  domain_name = ""
  if lb["Scheme"] == "internal":
    domain_name = "private-nlb.{}".format(
      DOMAIN_NAME
    )
  else:
    domain_name = "public-nlb.{}".format(
      DOMAIN_NAME
    )

  r53_client.change_resource_record_sets(
    HostedZoneId=HOSTED_ZONE_ID,
    ChangeBatch={
      "Changes": [
        {
          "Action": "UPSERT",
          "ResourceRecordSet": {
            "Name": domain_name,
            "Type": "A",
            "AliasTarget": {
              "HostedZoneId": lb["CanonicalHostedZoneId"],
              "DNSName": lb["DNSName"],
              "EvaluateTargetHealth": True
            }
          }
        }
      ]
    }
  )

def filter_ingress_tags(load_balancers):
  filtered = []
  for lb in load_balancers:
    if lb["Type"] == "network":
      arn = lb["LoadBalancerArn"]
      print("Found NLB: {}".format(arn))
      tags = elb_client.describe_tags(
        ResourceArns=[
          arn
        ]
      )
      for tag_desc in tags["TagDescriptions"]:
        for tag in tag_desc["Tags"]:
          if (
            tag["Key"] == "kubernetes.io/service-name" and
            tag["Value"] == "ingress-nginx/ingress-nginx"
          ):
            filtered.append(lb)

  return filtered

def clear_parameter_store():
  """
  If the NLB is removed, the parameter should be reset to its
  initial "None" value. This is picked up by CFN Conditions.
  """
  print("Storing \"None\" value in the parameter store.")
  ssm_client.put_parameter(
    Name=PARAMETER_KEY,
    Value="None",
    Type="String",
    Overwrite=True
  )


def store_arn_in_parameter_store(eks_nlb_arn):
  """
  Store the previously found ARN in the parameter store. Also
  outputs some additional information about the changing state of the
  parameter.
  """
  eks_nlb_arn_param = None
  print("Fetching ARN from the parameter store.")
  try:
    param = ssm_client.get_parameter(
      Name=PARAMETER_KEY
    )
    eks_nlb_arn_param = param["Parameter"]["Value"]
  except ssm_client.exceptions.ParameterNotFound:
    print("Parameter {} not found in parameter store.".format(
      PARAMETER_KEY
    ))

  if eks_nlb_arn == eks_nlb_arn_param:
    print("No change in ARN detected.")
  else:
    print("Change in ARN detected! Old: {}, new: {}".format(
      eks_nlb_arn_param,
      eks_nlb_arn
    ))

    print("Storing ARN in the parameter store.")
    response = ssm_client.put_parameter(
      Name=PARAMETER_KEY,
      Value=eks_nlb_arn,
      Type="String",
      Overwrite=True
    )
    print("Put parameter response:")
    print(response)
```

As you can see from the source code, we also update route53 records for the load balancer. When the load balancer is private this allows us to connect to load balancer from internal systems at a predictable DNS address, instead of the one generated by Kubernetes. If the load balancer is public, this DNS name can be used as the origin for a CloudFront distribution.

**Using the stored ARN in CFN**

By adding a SSM based parameter to our stacks, the ARN becomes available for use:
```json
"Parameters": {
  "EksNlbArn": {
    "Default": "eks-nlb-arn",
    "Type": "AWS::SSM::Parameter::Value<String>"
  },
```

For API Gateway, the VPC Link can use the NLB ARN like this:
```json
"ApiGatewayBackendVpcLink": {
  "Properties": {
    "Name": "public-api.mydomain.net",
    "TargetArns": [
      {
        "Ref": "EksNlbArn"
      }
    ]
  },
  "Type": "AWS::ApiGateway::VpcLink"
},
```

When creating a VPC Endpoint Service as described in challenge four, you can use the NLB ARN like this:
```json
"VpcEndpointService": {
  "Properties": {
    "AcceptanceRequired": false,
    "NetworkLoadBalancerArns": [
      {
        "Ref": "EksNlbArn"
      }
    ]
  },
  "Type": "AWS::EC2::VPCEndpointService"
},
```

**Making sure dependent resources are only deployed when the NLB is present**

In our initial CloudFormation deployment we already define our SSM parameter and set its initial value to "None":
```json
"EksNlbArnParameter": {
  "Properties": {
    "Name": "eks-nlb-arn",
    "Type": "String",
    "Value": "None"
  },
  "Type": "AWS::SSM::Parameter"
},
```

In our API Gateway or ALB stacks we add a condition:
```json
"NlbPresent": {
  "Fn::Not": [
    {
      "Fn::Equals": [
        {
          "Ref": "EksNlbArn"
        },
        "None"
      ]
    }
  ]
},
```

And finally, for the resources that require the NLB to be present, we use the condition:
```json
"Resources": {
  "AlbWafStack": {
    "Condition": "NlbPresent",
    ...
  }
}
```

This guarantees that the resource (in this case an entire stack) is only deployed when the SSM Parameter `eks-nlb-arn` has a value different than "None". This is only the case when our Lambda updated the SSM Parameter with a real NLB ARN. 

## Eighth challenge: pointing an ALB to the IP addresses of the VPC Endpoint's ENIs
It's relatively easy to deploy a VPC Endpoint Service and subscribed VPC Endpoint:
```json
"VpcEndpointService": {
  "Properties": {
    "AcceptanceRequired": false,
    "NetworkLoadBalancerArns": [
      {
        "Ref": "EksNlbArn"
      }
    ]
  },
  "Type": "AWS::EC2::VPCEndpointService"
},
```

```json
"VpcEndpoint": {
  "Properties": {
    "PrivateDnsEnabled": false,
    "SecurityGroupIds": [
      {
        "Ref": "AlbWafVpcEndpointFirewallSecurityGroup"
      }
    ],
    "ServiceName": {
      "Fn::Join": [
        ".",
        [
          "com.amazonaws.vpce",
          "eu-west-1",
          {
            "Ref": "VpcEndpointService"
          }
        ]
      ]
    },
    "SubnetIds": [
      {
        "Ref": "AlbWafVpcEndpointSubnetGroupSubnet"
      },
      {
        "Ref": "AlbWafVpcEndpointSubnetGroupSubnet2"
      },
      {
        "Ref": "AlbWafVpcEndpointSubnetGroupSubnet3"
      }
    ],
    "VpcEndpointType": "Interface",
    "VpcId": {
      "Ref": "SkeletonVpcId"
    }
  },
  "Type": "AWS::EC2::VPCEndpoint"
},
```

Unfortunately, this does not provide us with the IP addresses for the NICs. The AWS CFN [specification](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpcendpoint.html) for `AWS::EC2::VPCEndpoint` gives us the ENI interface IDs, however. We can use a custom resource to obtain the backing IP addresses:

```json
"VpcEndpointCustomResource": {
  "DependsOn": [
    "VpceEndpointCustomResourceFunctionFunction",
    "VpcEndpoint"
  ],
  "Properties": {
    "AwsAccountId": "123123123123",
    "AwsRegion": "eu-west-1",
    "ServiceToken": {
      "Fn::GetAtt": [
        "VpceEndpointCustomResourceFunctionFunction",
        "Arn"
      ]
    },
    "VpcEndpoint": {
      "Ref": "VpcEndpoint"
    },
    "VpcEndpointDnsEntries": {
      "Fn::GetAtt": [
        "VpcEndpoint",
        "DnsEntries"
      ]
    },
    "VpcEndpointNetworkInterfaceIds": {
      "Fn::GetAtt": [
        "VpcEndpoint",
        "NetworkInterfaceIds"
      ]
    }
  },
  "Type": "AWS::CloudFormation::CustomResource"
},
```

The Lambda function backing this uses `crhelper.py`, which can be obtained from the [AWS Labs Github repository](https://github.com/awslabs/aws-cloudformation-templates/blob/master/community/custom_resources/python_custom_resource_helper/crhelper.py). The function itself looks like this:
```python
import crhelper
import os
import boto3
import json

# initialise logger
logger = crhelper.log_config({"RequestId": "CONTAINER_INIT"})
logger.info('Logging configured')
# set global to track init failures
init_failed = False

try:
    # Place initialization code here
    logger.info("Container initialization completed")
except Exception as e:
    logger.error(e, exc_info=True)
    init_failed = e


def fetch_vpc_ip_addresses(event):
    nic_ids = event["ResourceProperties"]["VpcEndpointNetworkInterfaceIds"]
    ec2 = boto3.resource("ec2")
    ip_addresses = []
    for nic_id in nic_ids:
        nic = ec2.NetworkInterface(nic_id)
        ip_addresses.append(nic.private_ip_address)
    
    print("Found addresses: {}".format(ip_addresses))
    return ip_addresses


def do_execution(event):
    ip_addresses = fetch_vpc_ip_addresses(event)

    physical_resource_id = "VpcEndpointCustomResource"
    response_data = {
        "IpAddresses": ip_addresses
    }
    return physical_resource_id, response_data


def create(event, context):
    print("create event")
    return do_execution(event)


def update(event, context):
    print("update event")
    return do_execution(event)


def delete(event, context):
    print("delete event")
    return


def handler(event, context):
    """
    Main handler function, passes off its work to crhelper's cfn_handler
    """
    # update the logger with event info
    global logger
    logger = crhelper.log_config(event)
    return crhelper.cfn_handler(event, context, create, update, delete, logger,
                                init_failed)
```

AS you can see from the source code, this custom resource fetches the `VpcEndpointNetworkInterfaceIds` and returns the IP addresses for every ENI. These IPs can then be used in the ALBs target groups:

```json
"ApplicationLoadBalancerTargetGroup80": {
  "Properties": {
    "Matcher": {
      "HttpCode": "200,403"
    },
    "Port": 80,
    "Protocol": "HTTP",
    "TargetType": "ip",
    "Targets": [
      {
        "Id": {
          "Fn::Select": [
            0,
            {
              "Fn::GetAtt": [
                "VpcEndpointCustomResource",
                "IpAddresses"
              ]
            }
          ]
        },
        "Port": 80
      },
      {
        "Id": {
          "Fn::Select": [
            1,
            {
              "Fn::GetAtt": [
                "VpcEndpointCustomResource",
                "IpAddresses"
              ]
            }
          ]
        },
        "Port": 80
      },
      {
        "Id": {
          "Fn::Select": [
            2,
            {
              "Fn::GetAtt": [
                "VpcEndpointCustomResource",
                "IpAddresses"
              ]
            }
          ]
        },
        "Port": 80
      }
    ],
    "VpcId": {
      "Ref": "SkeletonVpcId"
    }
  },
  "Type": "AWS::ElasticLoadBalancingV2::TargetGroup"
},
```

With this solution, our ALB is completely managed through CloudFormation, allowing us to specify its SSL certificates, security groups and more. Additionally, this allows us to integrate a WAF into the ALB, which was the goal we were trying to achieve.

## Ninth challenge: deploying the tagged subnets for load balancers

As described above, the subnets for the load balancers need to be tagged for EKS to understand which type of load balancer to deploy where. In this chapter we will go through the steps required to do that.

**Deploying the EKS Cluster Masters**

To deploy the Masters for your EKS Cluster you define a Security Group, IAM Role and some Subnets. Then you define the EKS Cluster like below. We snipped a lot of supporting resources from the JSON below for readability, but the essential resource (the "AWS::EKS::Cluster") is complete.
```json
"Outputs": {
    "EksClusterControlPlaneSecurityGroupId": {
      "Value": {
        "Ref": "EksClusterControlPlaneSecurityGroup"
      }
    },
    "EksClusterResourceName": {
      "Value": {
        "Ref": "EksCluster"
      }
    }
},
"Resources": {
  "EksCluster": {
    "Properties": {
      "ResourcesVpcConfig": {
        "SecurityGroupIds": [
          {
            "Ref": "EksClusterControlPlaneSecurityGroup"
          }
        ],
        "SubnetIds": [
          {
            "Ref": "EksClusterSubnetGroupSubnet"
          },
          {
            "Ref": "EksClusterSubnetGroupSubnet2"
          },
          {
            "Ref": "EksClusterSubnetGroupSubnet3"
          }
        ]
      },
      "RoleArn": {
        "Fn::GetAtt": [
          "EksClusterRole",
          "Arn"
        ]
      },
      "Version": "1.10"
    },
    "Type": "AWS::EKS::Cluster"
  }
}
```

Because we use nested stacks, we need to define some outputs. These outputs will be used as parameters in other stacks.

**Deploying the subnets for the load balancers**

The CFN template below shows an example of a tagged subnet. The `EksClusterResourceName` parameter references the EKS Cluster above.
```json
"Parameters": {
  "EksClusterResourceName": {
    "Type": "String"
  }
},
"Resources": {
  "InternalEksElbSubnet": {
    "Properties": {
      "AvailabilityZone": {
        "Fn::Select": [
          0,
          {
            "Fn::GetAZs": ""
          }
        ]
      },
      "CidrBlock": "10.128.129.128/27",
      "Tags": [
        {
          "Key": "Name",
          "Value": "InternalEksElbSubnet"
        },
        {
          "Key": "kubernetes.io/role/internal-elb",
          "Value": 1
        },
        {
          "Key": {
            "Fn::Join": [
              "",
              [
                "kubernetes.io/cluster/",
                {
                  "Ref": "EksClusterResourceName"
                }
              ]
            ]
          },
          "Value": "shared"
        }
      ],
      "VpcId": {
        "Ref": "SkeletonVpcId"
      }
    },
    "Type": "AWS::EC2::Subnet"
  }
}
```

The example above deploys a subnet for an `internal` load balancer. For `internet-facing` load balancers you just need to change the tags.

## Conclusion
EKS is very powerful and super nice to use, but if you want to use it in conjunction with other AWS services there are quite some hoops to jump through. Luckily, nothing is impossible, although you have to get creative sometimes. We hope this blog post will help you achieve hosting your Kubernetes workloads on AWS.

If you have any questions or remarks regarding this article you can reach out to the author on Twitter: [@donkersgood](https://twitter.com/donkersgood).
