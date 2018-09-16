---
layout: post
title: AWS API Gateway types, use cases and performance
banner: /assets/posts/2018-09-13-API-Gateway-Types-Compared/banner.png
author: lvandonkersgoed

---
API Gateway is a service that allows you to manage access to all sorts of backend systems. Since its release in 2015, many new features and variants have been added. In this post we'll explore the differences, use cases and performance of the Edge Optimized, Regional and Private API Gateway.

# API Gateway use cases
API Gateway does what its name implies: it's an API that functions as a gateway to upstream services. Its primary use cases are:
* Allowing HTTP access to services without a native HTTP interface, such as Lambda, Kinesis, Rekognition and many more
* Versioning different APIs, allowing for backwards compatibility
* Centrally monitoring and managing traffic coming in to multiple services
* Presenting a unified API frontend for multiple services
* Throttling; in general or for specific users
* Authentication; through Cognito or your own identication provider

# General architecture
AWS API Gateway resides in an AWS-managed environment. As one of the Serverless services, Amazon manages its hosting, redundancy, scaling, patching, and so on. This means that the underlying technology is completely invisible - all you get is a management interface (Web console, AWS API or AWS CLI). However you choose to configure your API, it will be hosted by AWS. If you want to learn more about Serverless, check out my blog post on [Building a Slack Bot with Serverless Framework](/2018/08/16/Building-a-Slackbot-with-Serverless-Framework.html).

Because a picture speaks a thousand words, here is a visualisation:
![Architecture](/assets/posts/2018-09-13-API-Gateway-Types-Compared/architecture.png)

# The original API Gateway: Edge Optimized
At its release in July 2015 (read the [announcement](https://aws.amazon.com/blogs/aws/amazon-api-gateway-build-and-run-scalable-application-backends/)), API Gateway allowed access to Lambda and publicly available HTTP endpoints. At that time, Lambdas could not be placed in VPCs - this feature was [released](https://aws.amazon.com/blogs/aws/new-access-resources-in-a-vpc-from-your-lambda-functions/) in February 2016. In its initial version, API Gateway came paired with a CloudFront distribution. The combined API Gateway and CloudFront would later be called **Edge Optimized** API Gateway, refering to the Edge locations available in CloudFront.

![Edge Optimized](/assets/posts/2018-09-13-API-Gateway-Types-Compared/edge-optimized-1.png)

Important details to remember regarding the first version of the Edge Optimized API Gateway:
* It uses CloudFront distributions, but you *can't* edit the distribution. Adding a WAF, for example, is not possible.
* It can access other sources, but those sources have to be publicly accessible.

# Regional API Gateways
In November 2017, Amazon [introduced](https://aws.amazon.com/about-aws/whats-new/2017/11/amazon-api-gateway-supports-regional-api-endpoints/) the **Regional** API Gateway. This variant, as its name implies, is deployed and accessible in a single region. In other words: it's the Edge Optimized API Gateway without the CloudFront distribution.

![Regional](/assets/posts/2018-09-13-API-Gateway-Types-Compared/regional.png)

Benefits to this variant are:
* You can deploy your own CloudFront distribution in front of your API Gateway. This distribution *can* have a WAF.
* If your API Gateway is going to be accessed from the same AWS region, the Regional API Gateway will have less latency.

# Integration with Private VPCs
A big downside to API Gateway has alweays been that any HTTP backend behind the API Gateway needed to be publicly accessible. And because the public IP address of the API Gateway is unknown or unpredictable, IP whitelisting at the backend system was not a viable option. To solve this issue, Amazon [introduced](https://aws.amazon.com/about-aws/whats-new/2017/11/amazon-api-gateway-supports-endpoint-integrations-with-private-vpcs/) integration with private VPCs.

The connection between your API Gateway and your private resources needs two parts:
* A private Network Load Balancer (NLB) in your VPC in front of the resources you want to access
* A API Gateway VPC Link that points to that NLB

![VPC Link](/assets/posts/2018-09-13-API-Gateway-Types-Compared/vpc-link.png)

Hurray! With this feature, API Gateway no longer demands your resources to be public and the security of your VPC-based resources is greatly enhanced.

# Private API Gateways
The integration with private VPC endpoints is an awesome improvement. However, there is still one major issue regarding the security of API Gateway: it is always publicly available. Of course you can implement authentication, rate limiting, API keys, WAF and [resource policies](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-resource-policies.html), but this still feels like a hack when your API Gateway is only used by your VPC-based resources or over AWS VPN. To solve this issue, AWS [introduced](https://aws.amazon.com/blogs/compute/introducing-amazon-api-gateway-private-endpoints/) the **Private** API Gateway in June 2018.

The Private API Gateway is not publicly accessible or resolvable. Instead, it can only be accessed from within your VPC. To achieve this, AWS uses the [PrivateLink](https://aws.amazon.com/about-aws/whats-new/2017/11/introducing-aws-privatelink-for-aws-services/) technology. Diagram time!

![Private API](/assets/posts/2018-09-13-API-Gateway-Types-Compared/private-api.png)

AWS PrivateLink is an incredible addition to the AWS technology stack. It allows you to access many AWS Services (e.g. S3, KMS, SNS, CodeBuild and API Gateway) over a network interface in your own VPC. Take S3 as an example: before PrivateLink, any traffic from an EC2 instance to an S3 bucket would travel over the public internet. With PrivateLink, all traffic to S3 is routed to the Elastic Network Adapter(s) you've created in your own VPC. From this interface, the traffic will only traverse Amazon's internal network to S3. And as a bonus, the PrivateLink endpoint for S3 allows you to specify a policy which buckets can be accessed. This is a *great* security feature, because this effectively solves the problem of unauthorized data extraction to unknown buckets.

Anyway, we digress. Back to API Gateway. By configuring an API Gateway as Private, it gets assigned a DNS name that can only be resolved from within the VPC. When you resolve the hostname for your API Gateway, the IP addresses for the PrivateLink Network Adapters are returned. Through this, all traffic to your API Gateway is routed over the AWS network.

There is one caveat: if you try to access a Private API Gateway over VPN or VPC Peering, the hostname for the API Gateway will not resolve at the other end. There are two solutions for this:
* Use the VPC Endpoint's DNS, which is publicly resolvable. However, you do need to specify the `Host` Header. An example in cURL: `curl https://vpce-04dee3730b094b205-zg5cu7h2.execute-api.eu-central-1.vpce.amazonaws.com/test -H 'Host: oxppfikjm4.execute-api.eu-central-1.amazonaws.com'`
* Place an Application Load Balancer with an SSL certificate (e.g. `api.mydomain.com`) in front of the IP addresses of your PrivateLink network interfaces. Also deploy a custom domain name for `api.mydomain.com` and a base path mapping for your API Gateway. Then add a Route 53 record that points `api.mydomain.com` as an alias to your ALB. This solution is quite complex, but we've tested it and it works. We might write a separate blog post about it later.

# When to use which API Gateway
We've discussed the three variants of API Gateway: Edge Optimized, Regional and Private. The big question is when to use which. Coincidentially, we made a flow chart for this!

![Flowchart](/assets/posts/2018-09-13-API-Gateway-Types-Compared/flowchart.png)

# Performance
We've described that the three API Gateways have very different access methodologies. Edge Optimized API Gateway uses CloudFront, Regional API Gateway uses direct access over the Internet and Private API Gateway uses PrivateLink technology. These technologies will have an effect on the latency of the API Gateway. We have run performance tests on each of the API Gateway types from locations around the world. You will find the results below.

A visual overview of the connection patterns for the three variants:
![Connections](/assets/posts/2018-09-13-API-Gateway-Types-Compared/connections.png)

## Test methodology
We have created three API Gateways in the `eu-central-1` (Frankfurt) region; one of each type. All APIs have exactly the same setup, with one `GET` method on `/` which returns a mock HTML page with a simple "Hello World!" string. This reduces the test setup to API Gateway only, so no external system (like a backend Lambda or EC2 instance) can affect the results. Caching is disabled for all API Gateways.

We created EC2 instances in four different regions: `eu-central-1` (Frankfurt), `eu-west-1` (Ireland), `us-east-1` (North Virginia) and `sa-east-1` (São Paulo). Each of these instances will execute a large number of requests and calculate the average latency over those requests. To access the Private API Gateway from other regions than Frankfurt, VPC Peerings are deployed. All instances used in the test are of the `c5.large` type, to eliminate any throttling at the source (however unlikely).

## Test tool
We're using Apache Bench (`ab`) to test API Gateway latency. Our tests will execute 8000 requests with a concurrency of 100. 

An example test command (this one is used for the Private API Gateway): `ab -n8000 -c100 -H 'Host: oxppfikjm4.execute-api.eu-central-1.amazonaws.com' https://vpce-04dee3730b094b205-zg5cu7h2.execute-api.eu-central-1.vpce.amazonaws.com/test`

Because of the Edge locations, latency alone is not a good performance metric. If a user connects to an Edge location, its latency will most likely be very low, because the Edge location is geograpically close to the user. However, the *processing time* will be a lot longer, because the Edge location needs to connect to a region that might be on the other side of the world. Therefore we will not use the Connect, Processing or Waiting metrics, but instead use Apache Bench's Total metric, which measures the time from the start of the request until the complete delivery of the response.

## Test results
The raw data for each region can be found here:
* [Frankfurt](/assets/posts/2018-09-13-API-Gateway-Types-Compared/frankfurt.txt)
* [Ireland](/assets/posts/2018-09-13-API-Gateway-Types-Compared/ireland.txt)
* [North Virginia](/assets/posts/2018-09-13-API-Gateway-Types-Compared/virginia.txt)
* [São Paulo](/assets/posts/2018-09-13-API-Gateway-Types-Compared/saopaulo.txt)

Full request results (n=8000), in milliseconds:

|  Frankfurt    | Edge | Regional | Private |
|--------------:|:----:|:--------:|:-------:|
|  Min          |  18  |   8      |  19     |
|  Mean         |  60  |   91     |  60     |
|  **Median**   |**58**| **81**   |**66**   |
|  Max          |  161 |   319    |  184    |

|  Ireland      | Edge | Regional | Private |
|--------------:|:----:|:--------:|:-------:|
|  Min          |  31  |   90     |  96     |
|  Mean         |  62  |   105    |  130    |
|  **Median**   |**61**| **103**  |**104**  |
|  Max          |  226 |   214    |  25036  |

|  N. Virginia  | Edge  | Regional | Private |
|--------------:|:-----:|:--------:|:-------:|
|  Min          |  100  |   352    |  356    |
|  Mean         |  110  |   356    |  364    |
|  **Median**   |**104**| **355**  |**360**  |
|  Max          |  1114 |   560    |  23498  |

|  São Paulo    | Edge  | Regional | Private |
|--------------:|:-----:|:--------:|:-------:|
|  Min          |  236  |   901    |  903    |
|  Mean         |  252  |   910    |  916    |
|  **Median**   |**241**| **909**  |**915**  |
|  Max          |  766  |   1120   |  1918   |

## Conclusions
A few things immediately jump out:
1. Edge Optimized API Gateways are **fast**.
1. A Private API Gateway is generally as fast as a Regional API Gateway.
1. Private API Gateways over VPC peering are less stable than the other variants.
1. Connecting to a Regional API Gateway from the same region generated the lowest *minimal* response time, but in our test, Edge Optimized was still faster on average.

All in all, we find the results quite surprising! 

**Point 1**: We had not expected the Edge Optimized API Gateway to be so much more performant than the other two types. Based on this we would advise to use Edge Optimized over Regional in most use cases. If you are using Regional, do place your own CloudFront distribution in front of it.

We can't say for sure *why* Edge Optimized is so much faster. In the end, traffic from the Edge locations will still need to travel to the API Gateway. It's unlikely that these paths are faster than other internal Amazon traffic, such as VPC Peerings. Other possible reasons for the fast Edge Optimized API are:
* There is still some caching taking place, even though caching has been disabled.
* Edge Optimized API Gateway has some internal smarts that allow certain elements to be executed on the Edge locations. We've executed our tests using Mock responses, and these could very well be delegated.

But if either of these mechanisms are in place, we would expect the Edge Optimized tests to be equally fast from all different regions. And we're not seeing that either. In fact, Frankfurt is fastest, followed by Ireland, followed by North Virginia, followed by São Paulo, which confirms that distance plays a role.

**Point 2**: Regional and Private gateways are in almost all situations equally fast. We did not expect this, because the traffic for the Private API Gateway stays within the Amazon networks, which we expected to be a bit faster. Of course, because we were using EC2 instances in our tests, it's well possible that the traffic for the Regional API Gateway *also* stayed within the Amazon network, which might explain this finding.

**Point 3**: Only when testing the Private API Gateway over VPC Peering did we see failing connections (timeouts). We're not sure if VPC Peering or PrivateLink is to blame, but this might warrant further investigation. These timeouts are the cause of the very high Max values in the data.

**Point 4**: Amazon's [documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-basic-concept.html) indicates that for in-region communication, using a Regional API Gateway "bypasses the unnecessary round trip to a CloudFront distribution". Because of this, we expected to see significant improvements here, but our tests did not show them. However, accessing a Regional API Gateway from within the same Region did yield the lowest reponse time of all our tests at 8ms.

# Final word
We hope that you enjoyed this post, and that the difference in performance and use cases between the three types of API Gateway have become more clear. If you have any questions or remarks, give the author a shout on [Twitter](https://twitter.com/donkersgood). 
