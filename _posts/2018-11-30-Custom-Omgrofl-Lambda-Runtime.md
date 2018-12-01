---
layout: post
title: A Custom Omgrofl Lambda Runtime
banner: /assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/banner.png
author: lvandonkersgoed

---
This Re:Invent, Amazon Web Services introduced a number of very powerful new features to Lambda. These include layers, custom runtimes and the ability to execute Lambdas through an ALB. Now what could be a better way to demonstrate these functions then by deploying a custom Omgrofl runtime?

As all of you know, [Omgrofl](https://esolangs.org/wiki/Omgrofl) is an esoteric programming language. Using statements like `iz`, `iz uber`, `iz nope liek`, `wtf` and `lmao`, it allows you to write [Turing complete](https://esolangs.org/wiki/Turing-complete) applications. That being said.. you probably shouldn't.

In this blog post, we'll use Omgrofl to prove without a doubt that you can now use any programming language in Lambda.

## Step 1: build a binary
The Lambda runtimes require you to provide the binary that executes the source code. In our case, we can use the [Omgrofl compiler](https://github.com/mneudert/omgrofl-compiler) from Marc Neudert.

The compiler is going to run in Lambda, so that's the architecture we should build it in. Navigate to the [Lambda Execution Environment and Available Libraries](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html) page and launch the AMI mentioned there. Then SSH into it and download and unzip the compiler:
```
[ec2-user@ip-172-31-76-130 ~]$ curl -L https://github.com/mneudert/omgrofl-compiler/archive/master.zip > master.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   130    0   130    0     0    821      0 --:--:-- --:--:-- --:--:--   822
100  6439    0  6439    0     0  35443      0 --:--:-- --:--:-- --:--:-- 35443
[ec2-user@ip-172-31-76-130 ~]$ unzip master.zip 
Archive:  master.zip
e52ee64b6dd7b98e9a340e874223b1f2ac99609c
   creating: omgrofl-compiler-master/
 extracting: omgrofl-compiler-master/.gitignore  
  inflating: omgrofl-compiler-master/Makefile  
  inflating: omgrofl-compiler-master/README.md  
   creating: omgrofl-compiler-master/src/
  inflating: omgrofl-compiler-master/src/ast.cpp  
  inflating: omgrofl-compiler-master/src/ast.h  
  inflating: omgrofl-compiler-master/src/lexer.cpp  
  inflating: omgrofl-compiler-master/src/lexer.h  
  inflating: omgrofl-compiler-master/src/omgrofl.cpp  
  inflating: omgrofl-compiler-master/src/parser.cpp  
  inflating: omgrofl-compiler-master/src/parser.h  
[ec2-user@ip-172-31-76-130 ~]$ 
```

Move into the directory and make sure `gcc-c++` is installed by running `sudo yum install gcc-c++ -y`. Then build the compiler with `make`. This gets us the binary we need:
```
[ec2-user@ip-172-31-76-130 omgrofl-compiler-master]$ make
  CXX  src/lexer.cpp
  CXX  src/ast.cpp
  CXX  src/parser.cpp

  >>   ./omgrofl
> OK <
```
Copy the `omgrofl` executable to your local machine.

Now add a handler called `bootstrap` that will forward incoming requests to the Omgrofl executable:
```sh
#!/bin/sh

set -euo pipefail

while true
do
  HEADERS="$(mktemp)"
  # Get an event
  EVENT_DATA=$(curl -sS -LD "$HEADERS" -X GET "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/next")
  REQUEST_ID=$(grep -Fi Lambda-Runtime-Aws-Request-Id "$HEADERS" | tr -d '[:space:]' | cut -d: -f2)

  # Execute the handler function from the script
  echo "$_HANDLER"
  RESPONSE=$(/opt/omgrofl "$_HANDLER")
  generate_post_data()
{
  cat <<EOS
{
  "statusCode": 200,
  "statusDescription": "200 OK",
  "isBase64Encoded": false,
  "headers": {
    "Content-Type": "text/html; charset=utf-8"
  },
  "body": "$RESPONSE"
}
EOS
}

  echo $(generate_post_data)

  # Send the response
  curl -X POST -H "content-type: application/json" "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/response" --data "$(generate_post_data)"
done
```

Make sure these files are executable and zip them:
```
→ chmod 755 bootstrap omgrofl 

→ zip runtime.zip bootstrap omgrofl
updating: bootstrap (deflated 40%)
updating: omgrofl (deflated 70%)
```

## Step 2: create a runtime in the console
Navigate to the Lambda console and click Layers.
![Layers](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/layers.png)

Then click Create Layer, fill in the fields and upload your runtime zip.
![Create Layer](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/create_layer.png)

Don't forget to copy the ARN, you will need it in the next step.

## Step 3: creating an Omgrofl lambda function
Move to the Functions section of the Lambda console and create a new function.
![Create Function](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/create_function.png)

Click Create Function, then select the Layers configuration:
![Select Layers](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/select_layers.png)

Then click Add a layer and fill in the ARN we copied in the previous step.
![Add Layer](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/add_layer.png)

## Step 4: create the Omgrofl application
Unfortunately, the Lambda console does not allow us to write inline code for custom runtimes. Therefore we will need to create a deployment package containing our Omgrofl application. To do this, create a new file called `hellolambda.omgrofl` with the following content:
```
w00t a Hello, World! program by poiuy_qwert
lol iz 72
rofl lol
lol iz 101
rofl lol
lol iz 108
rofl lol
rofl lol
lool iz 111
rofl lool
loool iz 44
rofl loool
loool iz 32
rofl loool
loool iz 87
rofl loool
rofl lool
lool iz 114
rofl lool
rofl lol
lol iz 100
rofl lol
lol iz 33
rofl lol
stfu
```
Then zip this file by running `zip package.zip hellolambda.omgrofl` and uploading that file to your lambda. Don't forget to change the Handler to `hellolambda.omgrofl`. Now run the Lambda and you should get your first output!

![Run Lambda](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/run.png)

## Final step: create an ALB
Navigate to the Load Balancers section in the EC2 console and create a new ALB. Click next until you reach step 4: configure routing. Select the Lambda function target type in this screen. At step 5, select your Omgrofl Lambda function.

Complete the last few steps and wait for your ALB to be provisioned. Then navigate to the Load Balancer's DNS name and here you are:
![Webpage](/assets/posts/2018-11-30-Custom-Omgrofl-Lambda-Runtime/webpage.png)

Your very own Omgrofl powered website! I think we're at the brink of a new technology revolution; within a year at least 30% of all internet traffic will be powered by Omgrofl. And all of this has been made possible by AWS and it's awesome new Lambda Runtime API.

If you have any questions, or if you would like to share your own Lambda runtimes ([ArnoldC](https://github.com/lhartikk/ArnoldC), [Brainfuck](https://esolangs.org/wiki/Brainfuck) or [L33t](https://en.wikipedia.org/wiki/Leet_(programming_language)), anyone?), reach out to me on [Twitter]( https://twitter.com/donkersgood).
