---
layout: post
title: "Part 1: Integrating OpsWorks and CodeDeploy"
date: 2014-11-23 11:09:28 +1000
comments: true
categories:
 - Chef
 - OpsWorks
 - CodeDeploy
 - EC2
 - AWS
 - Integrating OpsWorks and CodeDeploy
---

Amazon [recently announced](http://aws.amazon.com/blogs/aws/code-management-and-deployment/) a new deployment service
called [CodeDeploy](http://aws.amazon.com/codedeploy/). [OpsWorks](http://aws.amazon.com/opsworks/) is another application
 management product which provides excellent configuration management via [Chef](https://www.getchef.com/), however it
lacks the advanced deployment functionality of CodeDeploy. It therefore makes sense to integrate these two products,
delegating the configuration management to OpsWorks and the deployment functionality to CodeDeploy.

This is part 1 of [integrating OpsWorks and CodeDeploy](/blog/categories/integrating-opsworks-and-codedeploy/).

This section provides an introduction to OpsWorks and CodeDeploy, and the basic configuration required to get started.

<!-- more -->

## Why not just use OpsWorks?

OpsWorks is a great product, but it lacks several key deployment features such as the ability to run rolling
 deployments and cancel an in-flight deployment.

## Why not just use CodeDeploy?

While CodeDeploy does support the execution of configuration scripts in lifecycle events, these could easily become
 difficult to maintain if your configuration is complex. Also, the configuration might not belong to any specific
 application, and if you're running multiple applications per instance it might make sense to configure certain
 shared services on a per-server basis rather than per-application. OpsWorks is an excellent solution to these issues
 as it supports Chef and per-instance setup and configuration lifecycle events.

# Getting started

To get started you'll need to set up an OpsWorks stack with a custom cookbook repository. The stack also must be created
 in a region where CodeDeploy is supported, such as North Virginia. If you're using a VPC don't forget to configure your
  VPC to [allow external connectivity](http://docs.aws.amazon.com/opsworks/latest/userguide/workingstacks-vpc.html).

My stack configuration is as follows, as you can see I'll be using Ubuntu 14.04, but the steps should be similar on
 Amazon linux.

{% img /images/posts/opsworks_codedeploy/stack.png %}

Also, create a layer for your application servers. For example I've created a PHP App Server layer. Don't forget to
 enable "Public IP addresses" under the networking options.

{% img /images/posts/opsworks_codedeploy/layer.png %}

In this example I'm also going to deploy from S3, rather than GitHub. Therefore I'll assume you have an S3 bucket created
to host the zip deployment packages.

# Creating a CodeDeploy Service Role

You'll need to create a service role for CodeDeploy before proceeding, although if you've already followed the "Sample
Deployment" wizard then you will probably have created one at the following step:

{% img /images/posts/opsworks_codedeploy/policywizard.png %}

If you need to create one manually then you can [follow these steps](http://docs.aws.amazon.com/codedeploy/latest/userguide/how-to-create-service-role.html)
 to first create a role with the following policy:

{% img /images/posts/opsworks_codedeploy/policy.png %}

Then set the trust relationships:

{% img /images/posts/opsworks_codedeploy/trust.png %}

# Creating CodeDeploy application

Next go to the CodeDeploy console and create a new application using the "Custom Deployment" option.

In the application options you have to define which EC2 instance tags will be included in the deployment. Set the <code>
opsworks:stack</code> and <code>opsworks:layer:php-app</code> to the name of your stack and layer respectively.

{% img /images/posts/opsworks_codedeploy/codedeploy_app.png %}

Select a Deployment Config (eg. <code>CodeDeployDefault.OneAtATime</code>), and set the Service Role ARN to the service
 role you created earlier.


## Integrating OpsWorks and CodeDeploy

* [Part 1](/blog/2014/11/23/integrating-opsworks-and-codedeploy/) - Introduction and getting started.
* [Part 2](/blog/2014/11/24/part-2-integrating-opsworks-and-codedeploy/) - OpsWorks configuration and recipes.
* [Part 3](/blog/2014/11/25/part-3-integrating-opsworks-and-codedeploy/) - Deployment and results.