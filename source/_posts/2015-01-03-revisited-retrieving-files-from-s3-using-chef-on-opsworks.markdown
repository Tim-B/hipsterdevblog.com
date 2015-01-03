---
layout: post
title: "Revisited: Retrieving Files From S3 Using Chef on OpsWorks"
date: 2015-01-03 09:13:42 +1000
comments: true
categories:
 - Chef
 - OpsWorks
 - AWS
 - S3
---

One of my earliest and most popular posts is [Retrieving Files From S3 Using Chef on OpsWorks](/blog/2014/06/22/retrieving-files-from-s3-using-chef-on-opsworks/).
 That posts uses the [Opscode AWS cookbook](https://github.com/opscode-cookbooks/aws) which in turn uses the [right_aws](https://github.com/rightscale/right_aws)
 gem. While this method is fine - particularly if you're not using OpsWorks - there are some situations where it's not ideal.

Recently I've started using the aws-sdk gem directly which is bundled with the OpsWorks agent. The version at the time of
 writing is 1.53.0.

The advantages of this are:

* Support for [IAM instance roles](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html),
 meaning you don't have to pass AWS credentials via your custom JSON.
* No dependencies on external cookbooks.
* Will ordinarily be run at the compile stage, therefore you could download a JSON file, parse it, then use it to
 generate resources if you wanted.

The disadvantages are:

* It's not entirely clear, but my feeling is that the gems included with the OpsWorks agent aren't necessarily
 part of the API "contract" provided by OpsWorks for cookbook developers. Therefore there is no guarantee that AWS
 won't change the version or even remove it entirely without notice. I think it's unlikely that they'll remove the
 aws-sdk gem or move to a version with compatibility breaking changes any time soon, but it's possible.
* Less "chef-like" solution, although you could write your own chef resource to wrap it
* If you're not using OpsWorks then the aws-sdk will create another dependency

This blog post provides an example of how to use the bundled aws-sdk gem to download a file from S3 using IAM instance
 roles on OpsWorks.

<!-- more -->

# The IAM role

Before starting you'll need to grant permissions to access your S3 bucket to the IAM role used by your instances. You can find
 and edit EC2 instance roles under Identity & Access Management > Roles. The default role will probably be called something
 like `aws-opsworks-ec2-role`.

If you need to find the role it unfortunately isn't listed in OpsWorks on the instance view page, but you can either
 find it in the instance properties within the EC2 console:

{% img /images/posts/s3_chef_revisited/ec2_role.png %}

Or in the security tab of the layer settings:

{% img /images/posts/s3_chef_revisited/layer_security.png %}

Add a new policy to this role granting permissions to access your desired S3 bucket, for example:

{% codeblock lang:json s3_access_policy %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:Get*",
        "s3:List*"
      ],
      "Sid": "Stmt0123456789",
      "Resource": [
        "arn:aws:s3:::my-bucket/*"
      ],
      "Effect": "Allow"
    }
  ]
}
{% endcodeblock %}

Don't forget to replace `my-bucket` with your bucket name. This will grant access to all the Get and List prefixed operations
 for all objects in the bucket, but you can use a more specific policy if desired.

# The recipe

I won't go into the details of creating a custom cookbook, S3 bucket or adding recipes to lifecycle events as these steps
 were covered in my [earlier post](/blog/2014/06/22/retrieving-files-from-s3-using-chef-on-opsworks/). Given that
 we won't be using the OpsCode AWS cookbook you can skip adding that to your Berskfile.

{% codeblock mycookbook/recipes/myrecipe.rb %}
require 'aws-sdk'

s3 = AWS::S3.new

# Set bucket and object name
obj = s3.buckets['my-bucket'].objects['my/object.txt']

# Read content to variable
file_content = obj.read

# Log output (optional)
Chef::Log.info(file_content)

# Write content to file
file '/tmp/myobject.txt' do
  owner 'root'
  group 'root'
  mode '0755'
  content file_content
  action :create
end
{% endcodeblock %}

Now if you run `mycookbook::myrecipe` you should find `/tmp/myobject.txt` is populated with the content of `my/object.txt`
 in the `my-bucket` bucket.

Note that as I mentioned earlier the aws-sdk version at the time of writing is 1.53.0, therefore remember to refer to the
**[SDK version 1 docs](http://docs.aws.amazon.com/AWSRubySDK/latest/frames.html)** rather than the version 2 docs which will
 use the `Aws.` namespace in examples (note the capitalization).

# Reading and parsing a JSON file to use in a recipe

Here's an example which downloads a JSON file, parses it then uses it to create resources.

{% codeblock fruits.json %}
{
	"fruit": [
		"apples",
		"oranges",
		"bananas"
	]
}
{% endcodeblock %}

{% codeblock mycookbook/recipes/myrecipe.rb %}
require 'aws-sdk'

s3 = AWS::S3.new

# Set bucket and object name
obj = s3.buckets['test-site-config'].objects['fruits.json']

# Read content to variable
file_content = obj.read

# Log output (optional)
Chef::Log.info(file_content)

fruits = JSON.parse(file_content)

fruits['fruit'].each do |fruit|
  log "I have #{fruit}" do
    level :info
  end
end
{% endcodeblock %}

In your OpsWorks run logs you should see something like:

{% codeblock %}
[2015-01-03T06:18:10+00:00] INFO: Processing log[I have apples] action write (mycookbook::myrecipe line 17)
[2015-01-03T06:18:10+00:00] INFO: I have apples
[2015-01-03T06:18:10+00:00] INFO: Processing log[I have oranges] action write (mycookbook::myrecipe line 17)
[2015-01-03T06:18:10+00:00] INFO: I have oranges
[2015-01-03T06:18:10+00:00] INFO: Processing log[I have bananas] action write (mycookbook::myrecipe line 17)
[2015-01-03T06:18:10+00:00] INFO: I have bananas
{% endcodeblock %}

# Compile vs. Converge time

Chef runs in two stages, the compile stage then the converge stage. In the first recipe the S3 code is run at compile
 time because it is not part of a resource, but actually writing it to a file
 using the file resource takes place at converge time.

Therefore in the following scenario:

{% codeblock mycookbook/recipes/myrecipe.rb %}
require 'aws-sdk'

s3 = AWS::S3.new
obj = s3.buckets['my-bucket'].objects['my/object.txt']
file_content = obj.read

file '/tmp/myobject.txt' do
  content file_content
  action :create
end

exists = File.file?('/tmp/myobject.txt')

log "File exists: #{exists}" do
    level :info
end
{% endcodeblock %}

The log will report `File exists: false`, as the file resource hasn't been executed by the time `File.file?` is called.

The consequences of this should generally be insignificant given that usually you'd want the S3 file to be downloaded first,
 however if you need to delay code to converge time for any reason you can simply wrap it in a [ruby_block resource](https://docs.chef.io/resource_ruby_block.html).












