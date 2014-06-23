---
layout: post
title: "Automatic DNS records using Route53 on OpsWorks"
date: 2014-06-23 18:28:23 +1000
comments: true
categories:
 - Chef
 - OpsWorks
 - Route53
 - EC2
 - DNS
---

<p>
    Lets say you have a load balanced web application managed with OpsWorks – your application traffic will be addressed
    to the load balancer, but sometimes it's still handy to address your application nodes directly for testing
    purposes or perhaps so each node has a unique SNS endpoint for HTTP notifications. You could just  use their IP,
    but unless you use an EIP that IP address may change. You could create a DNS record, which would be easier to remember
    and allows the IP to change – but managing this manually would be a pain.
</p>

<p>
    Fortunately this process of managing DNS records can be automated using Chef, Route53 and the EC2 instance
    metadata functionality to obtain the public IP. Each instance will automatically create an A record for
    <code>[instance name].example.com</code> on setup using their OpsWorks instance name.
</p>

<!-- more -->

<h2>Getting started</h2>
<p>
    Firstly, I'll assume you have a Route53 zone created – in my case the zone will be called <code>example.com</code>.
    You'll also need a set of AWS access keys, I recommend creating an IAM user restricted to managing your hosted zone.
    You can use the following IAM user policy:
</p>
{% codeblock lang:json IAM User Policy %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1403515694000",
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:GetHostedZone",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/<insert your hosted zone ID>"
      ]
    }
  ]
}
{% endcodeblock %}

<p>
    Next add your AWS credentials and zone ID as custom JSON variables in your OpsWorks stack:
</p>
{% codeblock lang:json Custom JSON %}
{
  "dns_zone_id"      : "<insert hosted zone ID>",
  "custom_access_key": "<insert access key>",
  "custom_secret_key": "<insert secret key>"
}
{% endcodeblock %}
<p>
 Adding custom JSON to your stack is covered in more detail in
 <a target="_blank" href="http://hipsterdevblog.com/blog/2014/06/22/retrieving-files-from-s3-using-chef-on-opsworks/">Retrieving Files From S3 Using Chef on OpsWorks</a>.
</p>
<p>
    Finally, add the <a target="_blank" href="http://community.opscode.com/cookbooks/route53">route53 cookbook</a>
    to your Berksfile. If you're not using Berkshelf you'll have to clone the whole cookbook into your custom cookbook
    repository.
</p>

{% codeblock lang:ruby Berksfile %}
source "https://api.berkshelf.com"

cookbook "route53", ">= 0.3.4"
{% endcodeblock %}

<h2>Creating your recipes</h2>
<p>
    Next, we need to create a custom cookbook and recipes – in this example the cookbook is called <code>dnsupdate</code>.
     Create the following file structure and files in your custom cookbook repository:
</p>

{% codeblock File Structure %}
dnsupdate/metadata.rb
dnsupdate/recipes/add.rb
{% endcodeblock %}

{% codeblock metadata.rb %}
name        "dnsupdate"
description "Update Route53 Zone"
maintainer  "Dilbert"
license     "Apache 2.0"
version     "1.0.0"

depends "route53"
{% endcodeblock %}

{% codeblock recipes/add.rb %}
require 'net/http'
include_recipe 'route53'

route53_record "create a record" do
  name  node[:opsworks][:instance][:hostname] + '.example.com'
  value Net::HTTP.get(URI.parse('http://169.254.169.254/latest/meta-data/public-ipv4'))
  type  "A"
  ttl   60
  zone_id               node[:dns_zone_id]
  aws_access_key_id     node[:custom_access_key]
  aws_secret_access_key node[:custom_secret_key]
  overwrite true
  action :create
end
{% endcodeblock %}

<p>
    Substitute <code>.example.com</code> with your own domain. <code>Net::HTTP.get(URI.parse('http://169.254.169.254/latest/meta-data/public-ipv4'))</code>
    is using the <a href="http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html">instance data API</a> to obtain the public IP
      – use the IP above any instance.
</p>

<h2>Adding recipe to life cycle event</h2>
<p>
    Once you've committed and pushed the new recipe to your custom cookbook repository you're ready to add the recipe
    to the configure life cycle event. First update your custom cookbooks, by going to your stack > Run Command > and
    selecting 'Update Custom Cookbooks' from the command select box.
</p>
<p>
    Finally, navigate to a layer in OpsWorks > Edit > Recipes > add 'dnsupdate::add' to the configure
    event and save.
</p>
{% img /images/posts/route53_dns/add_recipe.png %}
<p>
    Now when the run the configure event you should see a new DNS A record being added in Route53!
</p>