---
layout: post
title:  "Event Sourcing under steroids!"
date:   2016-12-31 08:42:27 +0100
categories: event sourcing aws lambda cqrs
---

I have started to play with [Amazon Web Services][amzon-web-services] (aka AWS) since some months. I found that AWS has made cloud easy. Really. 
It is easy to create your web server, to add a database, to manage network security, monitoring, supervision, ... They
have also introduced a really interesting product: [Lambda][aws-lambda] and the associated pattern: [Serverless architecture][serverless-applications].

Lambda is the representation of a pure function at server level. You have some inputs, you get some outputs, no state. So, it simplified the
scaling of your application. Deployment is fostered with the [serverless framework][serverless-framework]. With this tool, creating an
event sourcing application has never been so easy!

## Lambda

### What is it?

A lambda is just a small piece of code which complies to an interface. It can be written in Python, node.js
and java 8.

For Java, you have [differents way to create a lambda][java-lambda]. One way is to implement an interface:

{% highlight java %}
package org.test;

public class MyLambda implements RequestStreamHandler {
    @Override
    public void handleRequest(InputStream input, OutputStream output, Context context) throws IOException {
        // Your code goes here
    }
}
{% endhighlight %}

From there, you can access to others resources like RDS, DynamoDB, ...

### Using your lambda

There are differents ways to execute a lambda. Next are events that you can use to trigger or execute your lambda :
 - Directly from the AWS console
 - When a new DynamoDB record is created, ...
 - When a new message is created in your SNS topic
 - With API gateway
 - With SQS. In this case, you will need to use CloudWatch which will pool your queue and call your lambda when a new message is available
 - When something is inserted in your S3 bucket
 - Kinesis also

Lambdas seem to be the new key component of AWS.

### Deployment

You can found a complete example there. First, you need to define a role:
{% highlight bash %}
$ echo '{
    "Statement": [
        {
            "Action": "sts:AssumeRole",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Effect": "Allow"
        }
    ],
    "Version": "2012-10-17"
}' > role.json

$ aws iam create-role \
 --role-name handler-execution-role \
 --assume-role-policy-document file://role.json

$ aws iam attach-role-policy \
 --role-name handler-execution-role \
 --policy-arn  arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole 

$ aws iam attach-role-policy \
 --role-name handler-execution-role \
 --policy-arn  arn:aws:iam::aws:policy/service-role/AWSLambdaRole
{% endhighlight %}

Then, you need to create a deployment package. For java, you can find instructions [here][java-deployment-package].

Eventually, you can deploy your application with the AWS console or with the AWS cli:

{% highlight bash %}
$ aws lambda create-function \
  --function-name my-lambda \
  --zip-file fileb://build/distributions/mypackage.zip \
  --role arn:aws:iam::XXXXX:role/handler-execution-role  \
  --handler org.test.MyLambda \
  --runtime java8 \
  --timeout 15 \
  --memory-size 512

$ aws lambda update-function-code \
  --function-name my-lambda \
  --zip-file fileb://build/distributions/mypackage.zip 
{% endhighlight %}

You can also rely on the new [simplified serverless application model][aws-sam].

Quite simple, yes? But we can do better!


## Serverless framework

The serverless framework is designed to simplify lambdas deployment in order to create a complete application.
You can install it with npm:
{% highlight bash %}
$ npm install serverless -g
{% endhighlight %}

Then, you can start to create your new application:
{% highlight bash %}
$ serverless create --template aws-java-maven --path myApp
{% endhighlight %}

This command will create a new java-based lambda application. You can also do it in python, scala, ... [more info here][cli-ref-create],
even with the [go language][sparta].

At the root, you can find the serverless.yml file. It contains the description of your application, with the following groups:
 - service : your application name
 - provider : runtime (java, python or nodejs), region, stage (dev, test, production, ...)
 - functions : this is where you declare your lambdas
 - resources : associated resources like buckets, ... used by your lambdas

## Event sourcing with lambda and serverless framework

So let's start with event sourcing and see how lambda and serverless framework make it easy to create my application

### Event sourcing?

First of all, what is event sourcing? Event sourcing is an architecture popularized by [Greg Young][greg-young-blog].

As in every classical n-tier architecture, you have a huge database which keeps track of the current state of objects.
Most of time, only one big model has been designed representing the domain. More over, most of time, history of modifications
are not kept. For example, you can have your debt and credit of your bank account.

With event sourcing, you store events which modify the price. You determine the current state of the object by replaying 
events associated to this object. Using the same example with the bank account, we can have all the events on your 
accounts like withdrawing of deposing money.  Not only you can evaluate the state of your account by replaying
transactions, but you have the whole history in terms of business event (withdraw & deposit). You can use it
for future needs (for example, what are the average desposit & withdraw every month?), you can track strange 
events, you can prove why you are in this current state and so on. 

Event sourcing takes its root into DDD. To built it, you will need to find domains with their aggregated root and
work on each domain to extract business events.

Event sourcing can be though as a way to reveal the intention of your business by clearly using business events into 
every exchange in the application.

Here is a great [presentation of this subject][event-sourcing-presentation].

### Price-tracking application

![Architecture of an event sourcing application](/images/2016-12-31-event-sourcing-steroids-1.png){:class="img-responsive"}

#### Description

#### The read side

#### The write side

[amzon-web-services]: https://aws.amazon.com/
[aws-lambda]: https://aws.amazon.com/lambda/
[serverless-applications]: https://aws.amazon.com/lambda/serverless-architectures-learn-more/
[serverless-framework]: https://serverless.com/
[lambda-java]: http://docs.aws.amazon.com/lambda/latest/dg/java-programming-model-handler-types.html
[java-deployment-package]: http://docs.aws.amazon.com/lambda/latest/dg/create-deployment-pkg-zip-java.html
[cli-ref-create]: https://serverless.com/framework/docs/providers/aws/cli-reference/create/
[sparta]: https://github.com/mweagle/Sparta
[greg-young-blog]: https://goodenoughsoftware.net/
[event-sourcing-presentation]: https://www.youtube.com/watch?v=JHGkaShoyNs
[aws-sam]: http://docs.aws.amazon.com/lambda/latest/dg/deploying-lambda-apps.html
