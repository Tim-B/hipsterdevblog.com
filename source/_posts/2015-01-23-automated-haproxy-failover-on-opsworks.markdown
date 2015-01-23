---
layout: post
title: "Automated HAProxy failover on OpsWorks"
date: 2015-01-23 11:27:10 +1000
comments: true
categories:
 - HAProxy
 - High Availability
 - OpsWorks
 - SNS
 - Load Balancer
 - Failover
---

Without a doubt ELB is the simplest load balancing solution on AWS, however it may not be suitable for all users
 given it doesn't support features such as a static IP. Fortunately OpsWorks makes it only marginally more complicated
 to <a href="http://docs.aws.amazon.com/opsworks/latest/userguide/workinglayers-load.html">set up HAProxy</a> as an alternative.

The AWS ecosystem encourages you to implement redundancy across availability zones and to avoid a single point of
failure (SPOF). HAProxy will give you many additional features over ELB, however it is difficult to achieve cross-zone redundancy
 and automated failover as supported natively by ELB. DNS round-robbin can help balance load across multiple HAProxy instances
 to achieve scalability, however solution does not help to achieve high availability.

This blog post will demonstrate how to implement automated failover using a self-monitoring pair of HAProxy instances in an
 active/standby configuration. When a failure is detected the healthy standby will automatically take control of the
 elastic IP (EIP) assigned to the pair and ensure the service can continue to function. A notification will also be triggered
 via SNS to alert you that a failover has taken place.

<!-- more -->

# Getting started

I'll assume you're starting with a working OpsWorks stack including both an application server and HAProxy layer. I recommend
 disabling automatic assignment of an EIP in the HAProxy layer settings (make sure public address assignment is enabled though)
 then [manually register](http://docs.aws.amazon.com/opsworks/latest/userguide/resources-attach.html#resources-attach-eip) a single EIP
 with your stack and assign it to one of your HAProxy instances. I'll also assume you have a custom cookbook repository setup.
 Also, I'll assume you have a pre-existing SNS topic for failover notifications to be published to.

# Chef recipe

Create the following files within a custom cookbook:

{% codeblock lang:ruby haproxyfailover/recipes/setup.rb %}
service "monit" do
  supports :status => false, :restart => true, :reload => true
  action [:enable, :start]
end

template '/etc/failtome.sh' do
  source 'failtome.sh.erb'
  owner 'root'
  group 'root'
  mode 0755
end

template '/etc/monit.d/haproxywatch.monitrc' do
  source 'haproxywatch.monitrc.erb'
  owner 'root'
  group 'root'
  mode 0440
  notifies :reload, "service[monit]", :immediately
end
{% endcodeblock %}

{% codeblock lang:bash haproxyfailover/templates/default/failtome.sh.erb %}
#!/bin/sh

# Check current instance IP externally
OUT=$( curl -qSfsw '\n' http://checkip.amazonaws.com ) 2>/dev/null
RET=$?

# Check that current IP isn't target IP and that request didn't fail
if [ "$RET" = '0' -a "$OUT" != '<%= node[:stack][:primary_ip] %>' ]; then

 # Swap EIP
 aws --region <%= node[:opsworks][:instance][:region] %> opsworks associate-elastic-ip \
  --elastic-ip <%= node[:stack][:primary_ip] %> \
  --instance-id <%= node[:opsworks][:instance][:id] %>

 # Send notification
 aws --region <%= node[:opsworks][:instance][:region] %> sns publish \
  --topic-arn <%= node[:stack][:failover_topic] %> \
  --subject "HAProxy failover notification" \
  --message \
  "Instance <%= node[:opsworks][:instance][:id] %> took over <%= node[:stack][:primary_ip] %>"

fi
{% endcodeblock %}

{% codeblock haproxyfailover/templates/default/haproxywatch.monitrc.erb %}
check host <%= node[:stack][:primary_ip] %> with address <%= node[:stack][:primary_ip] %>
    if failed
        port 80
        protocol HTTP
        request /
        timeout 30 seconds
        for 3 cycles
        then exec "/etc/failtome.sh
{% endcodeblock %}

Add the `haproxyfailover::setup` recipe to the setup lifecycle event of your HAProxy layer.

# IAM permissions

Next you need to add additional policies to the EC2 IAM role used by your HAProxy instances. There's more details on
 finding your instance role in one of my [previous posts](/blog/2015/01/03/revisited-retrieving-files-from-s3-using-chef-on-opsworks/).

{% codeblock lang:json remap_eip %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1421894206000",
      "Effect": "Allow",
      "Action": [
        "opsworks:AssociateElasticIp"
      ],
      "Resource": [
        "arn:aws:opsworks:*:*:stack/<my stack ID>/*"
      ]
    }
  ]
}
{% endcodeblock %}

{% codeblock lang:json sns_alert %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1421894344000",
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": [
        "<my SNS ARN>"
      ]
    }
  ]
}
{% endcodeblock %}

Don't forget to replace `<my stack ID>` and `<my SNS ARN>` with your own values.

# Editing custom stack JSON

Edit your stack settings and add the following to your custom JSON.

{% codeblock lang:json custom_json %}
{
   "stack": {
      "primary_ip":"<my EIP>",
      "failover_topic":"<my SNS ARN>"
   }
}
{% endcodeblock %}

Don't forget to replace `<my EIP>` and `<my SNS ARN>` with your own values. Afterwards your custom JSON
 should look something like this:

{% img /images/posts/haproxyfailover/customjson.png %}

# Testing failover

After running the setup event again we're now ready to test our failover. You'll see that 54.173.141.243 is
 assigned to neptune initially:

{% img /images/posts/haproxyfailover/stage1.png %}

After stopping neptune 54.173.141.243 is still assigned to it because the failover hasn't triggered yet:

{% img /images/posts/haproxyfailover/stage2.png %}

Once the failover triggers 54.173.141.243 is now assigned to saturn:

{% img /images/posts/haproxyfailover/stage3.png %}

Also the following notification is received:

{% img /images/posts/haproxyfailover/stage4.png %}

Saturn will retain the IP even if neptune is brought back online, unless it is manually reassigned or saturn fails.

# Discussion

**Use of monit**

<a href="http://mmonit.com/monit/">moinit</a> is used on each instance to check that the primary IP is online. The
 OpsWorks agent itself uses monit and therefore it doesn't need to be installed on the instance. In the default OpsWorks
 configuration monit runs every 60 seconds. monit has a built in email alerting function, however in my opinion
 SNS is more manageable as alert recipients can easily be updated via the AWS console.

**Failover delay**

The monit configuration above specifies `for 3 cycles`, meaning that a failover won't be triggered unless
 3 consecutive monit runs fail. Therefore a failover could take about 4 minutes. This is an acceptable trade-off in most
 scenarios as a failover should be rare and you'd rather avoid failing over unnecessarily for an issue that resolves itself
 within a minute or two. You can however remove the `for 3 cycles` line to failover immediately if you'd prefer.

**Checking external IP**

To avoid the active instance attempting to failover to itself it first checks
 its IP against `http://checkip.amazonaws.com` when a failover occurs. An instance will only attempt to take over if the
 external IP returned isn't the primary IP. This also serves to verify the instance taking over has external connectivity,
 otherwise it might attempt to take over the IP when its own connectivity is causing the health check to fail.









