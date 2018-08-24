---
layout: post
title: Serverless Basic Authentication using a Custom Authorizer
banner: /assets/posts/2018-06-13-Serverless-Basic-Authentication-Custom-Authorizer/banner.png
github: svdgraaf/serverless-basic-authentication
author: svdgraaf
---

In a recent project, we needed our api's to be able to work with external systems. These systems only supported HTTP basic authentication (eg: username/password) for integrating with external systems.

Of course, Basic HTTP Authentication is the easiest and most straight forward way to integrate things and it's built in, in almost any HTTP client.

If you want to read more about basic authentication, I suggest you take a look at the [wikipedia page](https://en.wikipedia.org/wiki/Basic_access_authentication) or [RFC7617](https://tools.ietf.org/html/rfc7617).

API Gateway
-----------
HTTP Basic Authentication is a very simple standard, which is why almost everyone supports it. But not API Gateway. The easiest way to setup authentication on API Gateway is either to use API Keys or IAM authentication. Using IAM authentication makes you have to sign all requests. Using the API Keys, you need to add a header (`x-api-key` to each request, with the api key in it).

Fortunately, nowadays API Gateway supports custom authorizers. Also, it is now possible to configure API Gateway to get the api key from the custom authorizer. Which is everything we need to support basic authentication.

We have the user provide us with the Basic Authentication header (called `Authorization`). As a username, they pass the api key name, and as the password, they send the API key value. The client (browser) will do some magic (base64 encoding the username/password) and sends it along. We then pass the value of this header to the custom authorizer, which base64 decodes it, checks if the API key is valid, and forwards the API key value back to API Gateway. API Gateway will then use that API key and forward the context to the receiving endpoint (eg: Lambda function).

That looks like this:

![Custom HTTP Basic authorizer](/assets/posts/2018-06-13-Serverless-Basic-Authentication-Custom-Authorizer/api-gateway-http-basic-authorizer.png)

You can find the code for the custom authorizer on [Github](https://github.com/svdgraaf/serverless-basic-authentication/blob/master/basic_auth.py)

```python
# The custom authorizer gets the Authorization header in the incoming event
# the contents of the header will be in the authorizationToken key, eg:
# {
#     "type":"TOKEN",
#     "authorizationToken":"Basic Zm1vYmFy1ndNQ1loUGE4TUs5SlFWcGU3dVRqWDVGOEY1MUJXa0Q0YVVGZUI2MnQ=",
#     "methodArn":"arn:aws:execute-api:<regionId>:<accountId>:<apiId>/<stage>/<method>/<resourcePath>"
# }

def basicAuth(event, context):
    authorizationToken = event['authorizationToken']

    # we fetch the base64 encoded token from the header
    b64_token = authorizationToken.split(' ')[-1]

    # decode the base64 encoded header value
    username, token = base64.b64decode(b64_token).decode("utf-8").split(':')

    # check if the given api key actually exists
    client = boto3.client('apigateway')
    response = client.get_api_keys(nameQuery=username, includeValues=True)

    # if no keys found, deny access
    if len(response['items']) != 1:
        print("Couldn't find key")
        raise Exception('Unauthorized!')

    # if the key value does not match, deny access
    if response['items'][0]['value'] != token:
        print("Key value mismatch")
        raise Exception('Unauthorized!!')

    # All is well, return a policy which allows this user to access to this api
    # this call is cached for all authenticated calls, so we need to give
    # access to the whole api. This could be done by having a policyDocument
    # for each available function, but I don't really care :)
    arn = "%s/*" % '/'.join(event['methodArn'].split("/")[0:2])

    authResponse = {
        'principalId': username,
        'usageIdentifierKey': token,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': 'Allow',
                'Resource': arn
            }]
        }
    }
    print("Authentication response: %s" % authResponse)

    return authResponse
```

The real magic is in the response code. In the [AWS documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-lambda-authorizer-output.html) this is fairly poorly documented but the magic is in the `usageIdentifierKey` return value. When this key is present and API Gateway is configured to get the key value from the custom authorizer, API Gateway will use the value of this key as the api key. It needs to contain the *value* of the key, not the name, id or whatever, it's the actual value of the key (took me a while to figure that one out).

The `principalId` field is also forwarded in the event to the receiving lambda call of your api. You can also set additional context fields if needed, which are also forwarded to the receiving endpoint. You can find the whole spec in the [AWS Documentation page](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-lambda-authorizer-output.html) for it.

Plugin
------
I poured all of this into a [Serverless Basic Authentication](https://www.npmjs.com/package/serverless-basic-authentication) plugin. Which is ready to use and sets everything up for you. Neat!
