---
layout: post
title: "Part 3: Integrating OpsWorks and CodeDeploy"
date: 2014-11-25 11:43:36 +1000
comments: true
categories:
 - Chef
 - OpsWorks
 - CodeDeploy
 - EC2
 - AWS
 - Integrating OpsWorks and CodeDeploy
---

This is part 3 of [integrating OpsWorks and CodeDeploy](/blog/categories/integrating-opsworks-and-codedeploy/).

This section covers creating the CodeDeploy deployment, deploying it to the configured OpsWorks stack and demonstrating the results of the integration.
 [Click here](/blog/2014/11/23/integrating-opsworks-and-codedeploy/) for Part 1.

<!-- more -->

# Setting bucket policy and uploading deployment package

Now we're ready to deploy a package from CodeDeploy.

For the purposes of this example the package will contain two files:

{% codeblock File Structure %}
appspec.yml
app/index.php
{% endcodeblock %}

The [AppSpec](http://docs.aws.amazon.com/codedeploy/latest/userguide/app-spec-ref.html) specifies that the contents of the
 App directory should be copied to the location of the placeholder OpsWorks deployment.

{% codeblock lang:yml appspec.yml %}
version: 0.0
os: linux
files:
   - source: app
     destination: /srv/www/my_app/public
{% endcodeblock %}

{% codeblock lang:php app/index.php %}
<h1>Hello from CodeDeploy</h1>
{% endcodeblock %}

Zip this up and upload it to your S3 bucket where you'll store your deployment packages.

If you haven't already done so, you will also need to apply the S3 bucket policy provided [here](http://docs.aws.amazon.com/codedeploy/latest/userguide/how-to-deploy-revision.html)
 to allow the CodeDeploy role access to objects in this bucket:

{% img /images/posts/opsworks_codedeploy/bucket_policy.png %}

One important note is that you must also include the role ARN for your OpsWorks instances as above.

You can find the stack profile in the security settings of the layer:

{% img /images/posts/opsworks_codedeploy/instanceprofile.png %}

Then get the role ARN from your IAM console:

{% img /images/posts/opsworks_codedeploy/iaminstanceprofile.png %}

# Running CodeDeploy deployment

We're now ready to deploy our application with CodeDeploy, head back to the CodeDeploy application you created earlier
 and create a new deployment from the zip you uploaded.

{% img /images/posts/opsworks_codedeploy/codedeploy_deployment.png %}

Click "Deploy Now" and wait for the deployment to conclude.

Should your deployment fail, click "View All Instances" > "View Events" beside an instance and click "View Logs" beside the failed step.

## Viewing your application

Your application should now successfully be deployed to your OpsWorks instances. If you view the application in your
 browser you should see your deployment.

{% img /images/posts/opsworks_codedeploy/complete.png %}

## Caveats - Launching a new instance

Unfortunately it seems CodeDeploy currently only supports automatic deployments for new instances when they're in
 an autoscaling group. OpsWorks only supports its own load and time based instance functionality rather than
 using autoscaling groups, and therefore you'll have to manually trigger a deployment after a new instance comes online
 and before you place it under your load-balancer. You may wish for your recipes to cause the load-balancer health
 check to fail by default, then have a separate recipe which enables the health check to pass which you can run manually
 once you've run a deployment after a new instance has been launched.

Alternatively you could use your configure recipe to trigger a deployment automatically using the CodeDeploy API,
 however you would need to know which specific applications are relevant to the instance.

## Integrating OpsWorks and CodeDeploy

* [Part 1](/blog/2014/11/23/integrating-opsworks-and-codedeploy/) - Introduction and getting started.
* [Part 2](/blog/2014/11/24/part-2-integrating-opsworks-and-codedeploy/) - OpsWorks configuration and recipes.
* [Part 3](/blog/2014/11/25/part-3-integrating-opsworks-and-codedeploy/) - Deployment and results.