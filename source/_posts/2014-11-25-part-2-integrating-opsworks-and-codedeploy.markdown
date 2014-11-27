---
layout: post
title: "Part 2: Integrating OpsWorks and CodeDeploy"
date: 2014-11-24 11:43:29 +1000
comments: true
categories:
 - Chef
 - OpsWorks
 - CodeDeploy
 - EC2
 - AWS
 - Integrating OpsWorks and CodeDeploy
---

This is part 2 of [integrating OpsWorks and CodeDeploy](/blog/categories/integrating-opsworks-and-codedeploy/).

This section covers creating the OpsWorks Chef recipes to deploy your application via CodeDeploy.
 [Click here](/blog/2014/11/23/integrating-opsworks-and-codedeploy/) for Part 1.

<!-- more -->

# Installing CodeDeploy agent via Chef

Next we need to write a custom chef recipe to install the CodeDeploy agent and perform our desired configuration.

Checkout your cookbooks repository and create the following files:

{% codeblock File Structure %}
Berksfile
myrecipe/metadata.rb
myrecipe/recipes/agent-install.rb
myrecipe/recipes/vhost.rb
myrecipe/templates/default/myapp_vhost.erb
{% endcodeblock %}

Populate these files with the following:

{% codeblock lang:ruby Berksfile %}
source "https://supermarket.getchef.com"
{% endcodeblock %}

(We won't be using [Berkshelf](http://docs.aws.amazon.com/opsworks/latest/userguide/cookbooks-101-opsworks-berkshelf.html)
 in this tutorial, however you'll probably want to create this file any way if you're planning to extend this tutorial
  with your own configuration)

{% codeblock lang:ruby myrecipe/metadata.rb %}
name	'myrecipe'
recipe	'myrecipe::agent-install', 'Fetches, installs, and starts the AWS CodeDeploy host agent'
{% endcodeblock %}

This is the code which downloads, installs and starts the CodeDeploy agent service:

{% codeblock lang:ruby myrecipe/recipes/agent-install.rb %}
remote_file "#{Chef::Config[:file_cache_path]}/codedeploy-install.sh" do
    source "https://s3.amazonaws.com/aws-codedeploy-us-east-1/latest/install"
    mode "0744"
    owner "root"
    group "root"
end

execute "install_codedeploy_agent" do
  command "#{Chef::Config[:file_cache_path]}/codedeploy-install.sh auto"
  user "root"
end

service "codedeploy-agent" do
	action [:enable, :start]
end
{% endcodeblock %}

This code creates a directory for the CodeDeploy to deploy to, creates the vhost and enables it.

{% codeblock lang:ruby myrecipe/recipes/vhost.rb %}
directory '/srv/www/my_app/public/' do
  owner 'deploy'
  group 'www-data'
  recursive true
end

directory '/etc/apache2/sites-available/my_app.conf.d/' do
    owner 'root'
    group 'root'
    mode 0644
    recursive true
end

template 'myapp_vhost' do
    path  '/etc/apache2/sites-available/my_app.conf'
    owner 'root'
    group 'root'
    mode 0644
end

link "/etc/apache2/sites-enabled/my_app.conf" do
  to "/etc/apache2/sites-available/my_app.conf"
end
{% endcodeblock %}

This is a standard Apache vhost configuration based on the default OpsWorks template:
{% codeblock lang:apacheconf myrecipe/templates/default/myapp_vhost.erb %}
<VirtualHost *:80>
  ServerName myapp.com
  DocumentRoot /srv/www/my_app/public/

  <Directory /srv/www/my_app/public/>
    Options FollowSymLinks
    AllowOverride All
    Order allow,deny
    Allow from all
  </Directory>

  <Directory ~ "\.svn">
    Order allow,deny
    Deny from all
  </Directory>

  <Directory ~ "\.git">
    Order allow,deny
    Deny from all
  </Directory>

  LogLevel info
  ErrorLog /var/log/apache2/my_app-error.log
  CustomLog /var/log/apache2/my_app-access.log combined
  CustomLog /var/log/apache2/my_app-ganglia.log ganglia

  FileETag none

  RewriteEngine On
  IncludeOptional /etc/apache2/sites-available/my_app.conf.d/rewrite*

  IncludeOptional /etc/apache2/sites-available/my_app.conf.d/local*
</VirtualHost>
{% endcodeblock %}

In this step you may also like to create additional recipes for other configuration tasks, such as installing dependencies
or configuring your HTTP server.

# Adding recipes and packages to layer via OpsWorks

Once you've committed and pushed your recipes, go to OpsWorks and add the agent recipe to the configure lifecycle event
of your application server layer. Also add <code>ruby2.0</code> and <code>awscli</code> to the OS packages.

 {% img /images/posts/opsworks_codedeploy/recipes.png %}

# Creating placeholder deployment

Because OpsWorks doesn't perform certain default configuration tasks (such as creating a www-data group) until
 a deployment occurs it's easiest to create a placeholder OpsWorks deployment with a holding page which will be replaced
 by CodeDeploy. You could of course skip this step and manually configure everything via custom recipes.

In this instance we'll create a new repository for the placeholder which contains nothing but an index.php file containing
 a placeholder message.

{% codeblock lang:php index.php %}
<h1>This server is undergoing maintenance, please try reloading the page</h1>
{% endcodeblock %}

Create this deployment in OpsWorks:

{% img /images/posts/opsworks_codedeploy/opsworksdeploy.png %}

# Starting an instance

You can now start an instance in OpsWorks. It usually takes at least 20 minutes to boot and execute the setup and configure
 recipes. After this has complete your instance should have a status of online:

{% img /images/posts/opsworks_codedeploy/online.png %}

You should also see the placeholder message when you visit the IP in a browser:

{% img /images/posts/opsworks_codedeploy/holding.png %}

This should also have installed and started the CodeDeploy agent.

## Integrating OpsWorks and CodeDeploy

* [Part 1](/blog/2014/11/23/integrating-opsworks-and-codedeploy/) - Introduction and getting started.
* [Part 2](/blog/2014/11/24/part-2-integrating-opsworks-and-codedeploy/) - OpsWorks configuration and recipes.
* [Part 3](/blog/2014/11/25/part-3-integrating-opsworks-and-codedeploy/) - Deployment and results.