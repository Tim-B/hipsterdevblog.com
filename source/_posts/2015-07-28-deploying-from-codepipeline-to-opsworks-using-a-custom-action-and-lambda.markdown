---
layout: post
title: "Deploying from CodePipeline to OpsWorks using a Custom Action and Lambda"
date: 2015-07-28 20:40:42 +1000
comments: true
categories: 
 - CodePipeline
 - Lambda
 - OpsWorks
 - SNS
---

<p>
**Update: CodePipeline now has built-in support for both [Lambda](https://docs.aws.amazon.com/codepipeline/latest/userguide/how-to-lambda-integration.html)
 and [OpsWorks](https://docs.aws.amazon.com/opsworks/latest/userguide/other-services-cp.html). I'd now recommend using the built-in functionality
 rather than the method described in this post.**
<p>

Original post:

AWS recently released [CodePipeline](http://aws.amazon.com/codepipeline/) after announcing it last year. After doing the
[setup walkthrough](http://docs.aws.amazon.com/codepipeline/latest/userguide/getting-started-w.html) I was surprised
to see only the following deployment options!

-> {% img /images/posts/opsworks_codepipeline/deployopts.jpg %} <-

I'm sure other integrations are in the works, but fortunately
 CodePipeline supports [custom actions](http://docs.aws.amazon.com/codepipeline/latest/userguide/how-to-create-custom-action.html)
 allowing you to build your own integrations in the mean time.

If you were hoping to see this:

-> {% img /images/posts/opsworks_codepipeline/actionopts.png %} <-

Then read on!

I've implemented a custom action to deploy to OpsWorks using Lambda, you can find the full source of the Lambda
function [on GitHub](https://github.com/Tim-B/codepipeline-opsworks-deployer). It leverages the fact that CodePipeline
uses S3 to store artifacts between stages to trigger the Lambda function, and SNS retry behaviour for polling.

This blog posts explains how to configure your pipleine and the Lambda function to deploy to OpsWorks using CodePipeline.

<!-- more -->

# Overview

Here's a sequence diagram of how this Lambda powered CodePipeline action works:

-> {% img /images/posts/opsworks_codepipeline/codepipelineopsworks-diagram.png %} <-

# Getting started


## Create an initial pipeline

If you haven't done so already, I'd recommend creating an initial pipeline in CodePipeline. This will setup a bucket 
such as `codepipeline-us-east-1-1234567890` which you'll need to configure later.

## Create an SNS topic

You'll need an SNS topic which is used to poll both the CodePipeline task queue and the OpsWorks deployment status.

Once created, edit the "Topic Delivery Policy", go to "Advanced View" and set the following JSON policy:
 
{% codeblock lang:json %}
 {
   "http": {
     "defaultHealthyRetryPolicy": {
       "minDelayTarget": 30,
       "maxDelayTarget": 30,
       "numRetries": 20,
       "numMaxDelayRetries": 0,
       "numNoDelayRetries": 0,
       "numMinDelayRetries": 0,
       "backoffFunction": "linear"
     },
     "disableSubscriptionOverrides": false
   }
 }
{% endcodeblock %}

This will retry (poll) every 30 seconds up to 20 times (10 minutes).

## Create OpsWorks artifact bucket

Create a new S3 bucket where you'll store the final artifacts which will be deployed to OpsWorks. You could use the
 CodePipeline bucket, but I'd recommend making a separate one to simplify things. Mine is called `opsworks-artifacts`.
 
## Configure your OpsWorks stack

Create an OpsWorks stack with an application that deploys from an S3 archive in your OpsWorks artifact bucket.
The zip file name will be the name of the artifact in CodePipeline, I've called mine `opsworks.zip`.

-> {% img /images/posts/opsworks_codepipeline/opsworks_app.png %} <-

## Create Lambda IAM role

From the IAM console create a new role, select "AWS Lambda" as the type.

Skip adding a managed policy, but add the following policy as an inline policy:

{% codeblock lang:json %}
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:Put*"
         ],
         "Resource":[
            "arn:aws:s3:::{OpsWorks artifact bucket}/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "opsworks:CreateDeployment",
            "opsworks:DescribeDeployments",
            "opsworks:DescribeLayers",
            "opsworks:DescribeInstances"
         ],
         "Resource":[
            "{OpsWorks stack ARN}"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "SNS:Publish"
         ],
         "Resource":[
            "{SNS poll topic ARN}"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "codepipeline:PollForJobs",
            "codepipeline:PutJob*",
            "codepipeline:AcknowledgeJob"
         ],
         "Resource":[
            "*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "logs:*"
         ],
         "Resource":"arn:aws:logs:*:*:*"
      }
   ]
}
{% endcodeblock %}

Be sure to replace `{OpsWorks artifact bucket}`, `{OpsWorks stack ARN}` and `{SNS Topic ARN}` with appropriate values
for the resources created above.

## Create a new Lambda function

Create a new Lambda function from the Lambda console and select the "Hello World" template.

-> {% img /images/posts/opsworks_codepipeline/lambda_top.png %} <-

-> {% img /images/posts/opsworks_codepipeline/lambda_bottom.png %} <-

Select the IAM role you created above. I set the memory and timeout as 128MB and 30s, but keep in mind the Lambda
function needs to copy a zip file between buckets so you might need to increase this if you have a large artifact.

## Link SNS event source

Once you've created the lambda function, click the "Event sources" tab, click "Add event source" and link the SNS
topic you created above.

-> {% img /images/posts/opsworks_codepipeline/lambda_sns.png %} <-

You can leave it disabled for the moment.

## Subscribe SNS to events for CodePipeline bucket
 
Go to the S3 bucket automatically created by CodePipeline, it should be named like `codepipeline-us-east-1-1234567890`.
Under the events section of the properties, subscribe the SNS topic you created above:

-> {% img /images/posts/opsworks_codepipeline/snss3subscribe.png %} <-

## Create pipeline
 
Now you can actually create your pipeline in CodePipeline. You may need to create a dummy CodeDeploy stage otherwise
 the CodePipeline wizard won't let you finish:
 
-> {% img /images/posts/opsworks_codepipeline/empty_pipeline.png %} <-

# Create Custom Action

Custom actions don't seem to be editable via the console yet, so you'll have to add the action via the CLI.

Save the following to a file:

{% codeblock lang:json input.json %}
{
    "category": "Deploy", 
    "provider": "OpsWorks-Via-Lambda", 
    "version": "1", 
    "settings": {
    }, 
    "configurationProperties": [
        {
            "name": "Deploy-Bucket", 
            "required": true, 
            "key": false,
            "secret": false
        },
        {
            "name": "OpsWorks-Stack-ID", 
            "required": true, 
            "key": false,
            "secret": false
        },
        {
            "name": "OpsWorks-App-ID", 
            "required": true, 
            "key": true,
            "secret": false
        }
    ], 
    "inputArtifactDetails": {
        "minimumCount": 1, 
        "maximumCount": 1
    }, 
    "outputArtifactDetails": {
        "minimumCount": 0, 
        "maximumCount": 0
    }
}
{% endcodeblock %}

Then run the following command:

{% codeblock lang:bash %}
aws codepipeline create-custom-action-type --cli-input-json file://input.json
{% endcodeblock %}

Now when you reload the console and edit the pipeline you should be able to select the new `OpsWorks-Via-Lambda` deployment
 action in the list.
 
Fill out the details and save the pipeline:

-> {% img /images/posts/opsworks_codepipeline/action-settings.png %} <-

# Uploading Lambda function

Download or fork my Lambda function on GitHub: [https://github.com/Tim-B/codepipeline-opsworks-deployer](https://github.com/Tim-B/codepipeline-opsworks-deployer).

Open the `Gruntfile.js` and update the specified `lambda_deploy` arn value with the ARN of your lambda function.

Run an `npm install`.

Then run `grunt deploy`.

You should see something like:

```
Running "lambda_package:default" (lambda_package) task
codepipeline-opsworks-deployer@0.0.0 ../../../../../tmp/1438081544032.7822/node_modules/codepipeline-opsworks-deployer
Created package at ./dist/codepipeline-opsworks-deployer_0-0-0_2015-6-28-21-5-44.zip

Running "lambda_deploy:default" (lambda_deploy) task
Uploading...
Package deployed.
No config updates to make.

Done, without errors.
```


If you go back to the Lambda console you should see the code has been uploaded:

-> {% img /images/posts/opsworks_codepipeline/code_uploaded.png %} <-

Finally, go back to the "Event sources" tab of the Lambda and update the SNS source state to Enabled.

-> {% img /images/posts/opsworks_codepipeline/enable-sns.png %} <-

# Testing pipeline

You should now be ready to test the pipeline! Click "Release change".

After a minute or so your OpsWorks deployment should change to "In Progress":

-> {% img /images/posts/opsworks_codepipeline/deploying.png %} <-

You should soon see a new deployment in OpsWorks:

-> {% img /images/posts/opsworks_codepipeline/opsworks-status.png %} <-

When that changes to done:

-> {% img /images/posts/opsworks_codepipeline/opsworks_running.png %} <-

The status in CodePipeline should change to "Succeeded":

-> {% img /images/posts/opsworks_codepipeline/deploy_done.png %} <-

It's normal for there to be a 30 second - 1 minute delay between events showing up in CodePipeline due to the polling
involved.

## Failures

The Lambda function should pass most failures back to CodePipeline too, if you see a failure:

-> {% img /images/posts/opsworks_codepipeline/failed.png %} <-

Click the "Details" link and you should see a message:

-> {% img /images/posts/opsworks_codepipeline/failed_message.png %} <-

You can of course also check the Lambda status. By default it should also automatically fail deployments that take
longer than 9 minutes.

# Taking it further

The next step would be to add additional build or testing stages between Source and Production. For example, you could
deploy to a staging stack, then run load tests against the staging stack, before deploying to your production stack
if the load tests pass.

You should also be able to adapt this technique to other deployment and build processes besides OpsWorks. You could even
use CodePipeline to deploy your OpsWorks Chef code by adapting the Lambda function to create a Kitchen test action,
then a run setup command action.

As usual, feedback and pull request are welcomed!





 
 



 








 






