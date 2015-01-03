---
layout: post
title: "My 2015 AWS Wish List"
date: 2015-01-01 22:53:23 +1000
comments: true
categories:
 - AWS
 - CloudWatch
 - ELB
 - Route53
 - SNS
 - Lambda
---

As a new year dawns it occurred to me how much AWS functionality I now use heavily wasn't available only a year ago.
 Almost every day I check the [AWS blog](http://aws.amazon.com/blogs/aws/) to find some new feature is available. This
 got me thinking about the functionality I'd like to see in 2015, so I put together a list of my top 5.

I'm sure the engineers at AWS are already working on some (if not most) of these, but if not then hopefully someone sees
 this post and gets a great idea!

<!-- more -->

# #1 - EIPs for ELB

Elastic load balancer (ELB) doesn't support static IPs, instead you must CNAME your domain to the hostname for your ELB or
 use the special alias record in Route53. I understand why they preferred to design it this way - it's much easier to load
 balance the load balancing instances via DNS rather than via network routing. This solution works fine in the majority of scenarios,
 but unfortunately it makes things much more difficult if you need to give out a static IP to your customers for them to use
 in their A records.

Being able to assign an Elastic IP (EIP) to an ELB would eliminate a huge barrier for a lot of people. Scaling could be addressed
 with anycast to provide a virtual IP which addresses multiple load balancing instances. This would also help ELB to compete
 against [rackspace](http://www.rackspace.com.au/cloud/load-balancers/compare) and [Google Cloud](https://cloud.google.com/compute/docs/load-balancing/http/cross-region-example)
 which both offer this feature.

# #2 - A managed scheduling / timer service

Invoking global jobs on a schedule can be quite a hassle in distributed systems. On one hand you can't have a CRON job running
 on every instance because then you'll end up with multiple invocations, but on the other hand if you have just one instance
 that handles scheduling you need to ensure the job is triggered even if that instance fails or is replaced.

One solution is to use AWS Data Pipeline to start an instance and invoke a command that sends a message to SNS,
although starting an EC2 instance just to do this is quite expensive. Also, the minimum period is 15 minutes, and you don't have control
 down to the minute or second as to exactly when your task will be invoked.

A managed scheduling service that allows you to submit messages to an SNS topic would be great! SNS already supports
 message fan-out to queues and delivery retires so the only component missing is something to submit those messages
 at a specified time. Azure has a [similar solution](http://azure.microsoft.com/en-us/services/scheduler/) already.

# #3 - Search for CloudWatch logs

CloudWatch logs provides a convenient way to manage logs across multiple instances without having to leave the AWS ecosystem.
 It's still a new product, but it feels like it hasn't yet achieved its full potential.

Currently you can store logs and create metrics based on certain patterns appearing in your logs, but there's no search
other than the ability to filter by time!
 You can't archive your logs to S3 for external processing either, and the GetLogEvents API function is limited to 10,000
 logs per request and can only be called 10 times/second.

These limitations mean most users will probably have to use a second log aggregation service to cater for ad-hoc
 log searches and extraction. However, a search function would make a huge difference and enable CloudWatch to compete
 with the likes of Logentries and Loggly. Even integration with CloudSearch would be enough for users
 who have a large enough log volume to justify a dedicated search instance.

# #4 - HTTP endpoint triggers for Lambda functions

It's easy to think of opportunities where the new Lambda service could help "glue" different systems together. Unfortunately
 we're currently limited by how a Lambda function can be invoked - it either has to be done manually via the AWS API
 or via S3, DynamoDB or Kinesis events.

Lambda would become significantly more useful if functions could be triggered asynchronously with a simple REST endpoint
 without authentication. Obviously a lack of authentication isn't ideal, but it'd make it much easier to integrate
 with 3rd parties that support web-hook functionality. Imagine being able to trigger Lambda functions
 using a BitBucket commit hook, or a stored email notification from Mailgun.

# #5 - Zone tagging in Route53

Resource tags can be used in IAM policies to restrict users to particular tags. This is useful when creating accounts
 that can only access EC2 instances tagged as belonging to a specific department for example.

Users who manage DNS on behalf of some of their customers via Route53 would appreciate giving their customers
 direct access via an IAM account to manage their zones directly. This would help to cut down on support costs
 associated with making DNS updates on behalf of the customer due to changes unrelated to the product they provide.

You can currently restrict IAM accounts to a list of zone IDs, but maintaining this list is impractical when some customers
 have dozens of zones and several users that change regularly. It would make things much easier if zones could be tagged
 with a particular customer, then accounts can be limited to zones tagged with that customer.

# Worth a mention

Here are some other features I'd love to see but didn't make it into my top 5.

**Rolling deployments for OpsWorks** - ElasticBeanstalk and CodeDeploy both support rolling deployments, it's a shame
 OpsWorks is the odd one out!

**Trigger Lambda functions with SQS and SNS** - Being able to process SQS and SNS messages with Lambda would be great too,
 although if you could trigger a Lambda function with a HTTP endpoint then you could subscribe that to SNS instead.

**HTTP request routing via ELB** - I can live without this, but many users would find it useful to route certain paths
 to different sets of back end instances. [Google Cloud](https://cloud.google.com/compute/docs/load-balancing/http/content-based-example)
 supports this already.

So, what's on your wish list?




