# Distributed Tracing example

Here, we demonstrate trace context propagation for Distributed Tracing, in several non-trivial configurations.

- We start with a static HTML page, served by a Python Lambda function, and instrumented using the New Relic Browser
  Agent. It presents a text box, and submits the text via an HTTP POST to an API Gateway proxy endpoint, backed by that
  same Python Lambda function.
- In response to the POST, the Python Lambda splits the message into words, and sends each word as a message to an SQS
  queue, propagating the trace context in the SQS message headers.
- A NodeJS Lambda function is triggered by the SQS Queue. It processes messages, and picks up the Trace Context from the
  headers. Each SQS message is produced to an SNS Topic, and the trace headers are propagated.
- A Java Lambda function is triggered by the SNS topic. It accepts the trace context, and simply logs the messages.

Each function (and the Browser Agent) produces spans, which are sent to New Relic. In general, Lambda telemetry is
buffered, and sent to New Relic during some _subsequent_ invocation. So, depending on how the buffering works out, some
spans will be sent before others, and the trace may be temporarily missing some data, until all the Lambda function
instances are either invoked an additional time, or shut down.

## Building and deploying

### Prerequisites

- The [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Docker](https://docs.docker.com/get-docker/)
- The [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- [newrelic-lambda](https://github.com/newrelic/newrelic-lambda-cli#installation) CLI tool

Make sure you've run the `newrelic-lambda install` command in your AWS Region, and included
the `--enable-license-key-secret` flag.

### deploy script

From a command prompt, in this directory, run

    ./deploy.sh <accountId> <trustedAccountId> <region>

where `<accountId>` is your New Relic account ID, and  `<region>`
is your AWS Region, like "us-west-2". `<trustedAccountId>` is likely the same as your `<accountId>`. For users whose New
Relic account is a sub-account of another account, it's the parent account.

This will package and deploy the CloudFormation stack for this example function.

At this point, you can invoke the function. As provided, the example function doesn't pay attention to its invocation
event. If everything has gone well, each invocation gets reported to New Relic, and its telemetry appears in NR One.

### Invoking the application

This example is driven from the browser. The CloudFormation stack you've deployed has an Output value named 
`PythonSqsSenderApi`. This is the URL you should visit with your web browser to get started. 

On that page, enter one or more words into the form field, and hit submit. The words will be processed by the 
application, and spans will be generated for the various processing steps.

New Relic batches telemetry, and in the Lambda environment, we can only send a batch during a subsequent invocation. 
So, you'll likely need to submit the form a second time around ten seconds after your first submission to get all the 
telemetry from the first submission sent to New Relic. Otherwise, you may temporarily see partial traces, or notice 
missing spans.

## Code Structure

Now is also a good time to look at the structure of the example code.

### template.yaml

This application is deployed using a SAM template, which is a CloudFormation template with some extra syntactic sugar
for Lambda functions. In it, we tell CloudFormation where to find the code for our three functions, how to configure the
functions, and we express relationships between our functions and their resources, which lets the SAM template processor
manage the IAM permissions that allow SQS to invoke our Node function, and our Python function to create messages on
thet SQS queue.

We have one template for an application with three lambda functions, written in different languages. The SAM CLI manages
the build for all three functions, based on the `CodeUri` property pointing to the source root for the function on disk.

By managing all our application resources in a single stack, we simplify setup and teardown, and keep the references
between our resources in sync.

The template has a single entry in the `Outputs` section. This provides the URL for the API Gateway endpoint that we'll
use to interact with the application. You should visit that URL with a web browser after deploying the example
application.

### app.py

The Python lambda is invoked by our API Gateway proxy. It plays two roles here: it responds to GET requests with a
simple single-page application (instrumented with the New Relic Browser Agent), and it processes POST requests by
producing messages to an SQS queue.

The Python Agent automatically picks up trace headers from the API Gateway Proxy request. There are several ways to
configure API Gateway to invoke a Lambda function; this one is the most convenient, and requires no extra work to
recover the trace context generated by the Browser Agent.

We don't have automatic trace context propagation for most AWS services. Instead, you can easily attach the trace
context to SQS messages as attributes, as seen here. There are different ways to approach this: it's important to pick
one and stick with it within your application, so that your various resources can easily recover and propagate trace
contexts across AWS's various services.

### sqs-sns-bridge.js

The NodeJS lambda is invoked by SQS with a batch of messages. It recovers the trace context by decoding the attribute
that the Python lambda added, and produces SNS messages, propagating the trace context in a similar way.

### App.java

The Java 11 lambda is invoked by SNS with a batch of SNS topic messages. It simply logs the messages, after recovering
the trace context.

Note that we create a new subclass of `LambdaTracing` to recover our custom trace context. The
`SNSEventLambdaTracing` class is an example of how you can recover trace contexts in Java lambda functions. Out of the
box, Java's OpenTracing agent has only the most minimal support for trace context recovery. Normally, you'll need to
take a similar approach to this one, to fetch the trace context from the invocation event and decode it for the
OpenTracing API. 
