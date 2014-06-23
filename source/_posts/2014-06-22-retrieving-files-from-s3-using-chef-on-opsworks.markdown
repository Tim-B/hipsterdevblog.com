---
layout: post
title: "Retrieving files from S3 using Chef on OpsWorks"
date: 2014-06-22 17:44:02 +1000
comments: true
categories:
 - Chef
 - OpsWorks
 - AWS
 - S3
---

<p>
    Say you wanted to manage some configuration file in your OpsWorks stack – typically you'd create a custom Chef recipe,
    make your configuration file a template and store it within your custom cookbook repository. This approach works well
    in most instances, but what if the file is something not suited to version control such as a large binary file or
    perhaps a programmatically generated artifact of your system?
</p>
<p>
    In these cases you may prefer to store the file in an S3 bucket and automatically download a copy of the file
    as part of a custom recipe. In my case I wanted to have a dynamically generated (by a separate system)
    vhost configuration file which could be deployed to a stack using a simple recipe.
</p>
<!-- more -->
<h2>Adding AWS cookbook via Berkshelf</h2>

<p>
    The first thing you'll need to do is add the OpsCode <a target="_blank" href="http://community.opscode.com/cookbooks/aws">AWS
    cookbook</a> to your Berkfile. Note that Berkshelf is only supported on Chef 11.10 or higher on OpsWorks, so if your
    OpsWorks stack has an older version selected you'll have to either upgrade or include the whole AWS cookbook in your
    custom cookbook repository.
</p>
<p>
    If you don't already have a Berkfile you'll need to create one in your custom cookbook repository, otherwise simply
    add the AWS cookbook. Your Berkfile should look something like this:
</p>
{% codeblock lang:ruby Berksfile %}
source "https://api.berkshelf.com"

cookbook "aws", ">= 2.2.2"
{% endcodeblock %}

<h2>Creating an S3 bucket and a user which can access it</h2>
<p>
    You can probably figure out how to create a bucket on your own. In my case I have a bucket called 'test-site-config'
    and a file in there called 'vhost.map' which I want to download via Chef.
</p>
 {% img /images/posts/s3_chef_opsworks/bucket.png %}
<p>
    Next you'll need some AWS credentials for Chef to use while downloading the file. You can use your root account
    but I'd strongly suggest using an IAM user limited to your bucket instead. If you create a new IAM user you can
    use the following policy which will only permit reading objects from the specified S3 bucket (obviously replace
    'test-site-config' with your own bucket name:
</p>
{% codeblock lang:json IAM policy %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1403407152000",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::test-site-config/*"
      ]
    }
  ]
}
{% endcodeblock %}

<h2>Passing AWS credentials via custom JSON</h2>
<p>
    Now navigate to your stack in the OpsWorks console, click 'Stack Settings' then 'Edit' and modify the Custom JSON
    field to include variables for your access and secret key. If you already have custom JSON values then you'll
    need to merge the new values with your existing JSON, otherwise you can use the code below:
</p>
{% codeblock lang:json Custom JSON %}
{
  "custom_access_key": "<insert access key>",
  "custom_secret_key": "<insert secret key>"
}
{% endcodeblock %}
{% img /images/posts/s3_chef_opsworks/edit_stack.png %}

<h2>Creating your custom recipe</h2>
<p>
    In this instance I'll create a new recipe called 'deployfile' which does nothing but download my file and save it to the specified
    location, however you could just as easily include this code within an existing recipe.
</p>
<p>
    Create the following file structure and use the code below in your custom cookbook repository:
</p>
{% codeblock File Structure %}
deployfile/metadata.rb
deployfile/recipes/default.rb
{% endcodeblock %}

{% codeblock metadata.rb %}
name        "deployfile"
description "Deploy File From S3"
maintainer  "Dilbert"
license     "Apache 2.0"
version     "1.0.0"

depends "aws"
{% endcodeblock %}

{% codeblock recipes/default.rb %}
include_recipe 'aws'

aws_s3_file "/etc/apache2/vhost.map" do
  bucket "test-site-config"
  remote_path "vhost.map"
  aws_access_key_id node[:custom_access_key]
  aws_secret_access_key node[:custom_secret_key]
end
{% endcodeblock %}

<p>
    Substitute <code>/etc/apache2/vhost.map</code> with the destination on your nodes, the bucket name and the remote
    path as required. You can also use other attributes belonging to the <a target="_blank" href="http://docs.opscode.com/resource_file.html">Chef
    file resource</a>.
</p>

<h2>Updating stack and executing recipe</h2>
<p>Once the code above has been committed and pushed back to your repository you're finally ready to execute the recipe.</p>

<p>Go to your stack and click 'Run Command', select 'Update Custom Cookbooks':</p>
{% img /images/posts/s3_chef_opsworks/update_cookbook.png %}
<p>Once OpsWorks has finished updating your custom cookbooks go back to 'Run Command' and select 'Execute Recipes'.
Enter the name of your recipe into the 'Recipes to execute' field:</p>
{% img /images/posts/s3_chef_opsworks/deploy_file.png %}
<p>Alternatively you can add your recipe to a layer life-cycle event (such as setup) and execute that life-cycle event
instead</p>
<p>Once that recipe has finished executing the file downloaded from S3 should now be present on your system!</p>