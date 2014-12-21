---
layout: post
title: "Monitoring per Application metrics with CloudWatch logs and OpsWorks"
date: 2014-12-21 10:53:24 +1000
comments: true
categories:
 - OpsWorks
 - CloudWatch
 - logs
 - Apache
 - Chef
---

[CloudWatch logs](http://aws.amazon.com/about-aws/whats-new/2014/07/10/introducing-amazon-cloudwatch-logs/) is a cheap and
 easy to set up centralised logging solution. At the moment it lacks several valuable features such as a convenient way
 to search logs, however it does an *excellent* job at providing graphing and alerting on aggregated metrics pulled from
 ingested log data. An obvious application for this is to monitor HTTP server statistics to provide graphs of overall
 request rates, response sizes, and error rates.


[OpsWorks](http://aws.amazon.com/opsworks/) makes it easy to orchestrate a fleet of EC2 instances serving multiple applications
 (as oppose to [Elastic Beanstalk](http://aws.amazon.com/elasticbeanstalk/) which only hosts a single application). Apache is
 the default HTTP server for most OpsWorks layer types.

This post demonstrates how to setup CloudWatch logs for Apache access logs on OpsWorks, then create custom CloudWatch
 metrics for an individual OpsWorks application to graph the HTTP request rate.

<!-- more -->

# Installing the CloudWatch agent with Chef to monitor Apache logs

The first step is to install the CloudWatch agentusing a custom recipe. These instructions are based off [the AWS documentation](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/QuickStartChef.html)
 so **follow those steps to configure your IAM instance role first**.

Create the following files in your custom cookbooks repository, you can name the custom recipe anything you like but
 in this example I've named it `myrecipe`.

{% codeblock myrecipe/recipes/logging.rb %}
template "/tmp/cwlogs.cfg" do
  source "cwlogs.cfg.erb"
  owner "root"
  group "root"
  mode 0644
end

directory "/opt/aws/cloudwatch" do
  recursive true
end

remote_file "/opt/aws/cloudwatch/awslogs-agent-setup.py" do
  source "https://s3.amazonaws.com/aws-cloudwatch/downloads/latest/awslogs-agent-setup.py"
  mode "0755"
end

execute "Install CloudWatch Logs agent" do
  command "/opt/aws/cloudwatch/awslogs-agent-setup.py -n -r us-east-1 -c /tmp/cwlogs.cfg"
  not_if { system "pgrep -f aws-logs-agent-setup" }
end
{% endcodeblock %}

{% codeblock myrecipe/templates/default/cwlogs.cfg.erb %}
[general]
# Path to the AWSLogs agent's state file. Agent uses this file to maintain
# client side state across its executions.
state_file = /var/awslogs/state/agent-state


[<%= node[:opsworks][:stack][:name] %>-http-access]
datetime_format = [%Y-%m-%d %H:%M:%S]
log_group_name = <%= node[:opsworks][:stack][:name].gsub(' ','_') %>-http-access
file = <%= node[:apache][:log_dir] %>/*-access.log
log_stream_name = <%= node[:opsworks][:instance][:hostname] %>
{% endcodeblock %}

The significant line is `file = <%= node[:apache][:log_dir] %>/*-access.log` which
 sets the log location to the Apache HTTP access logs.

Next, add this recipe to the setup lifecycle event of your OpsWorks layer:

{% img /images/posts/cwlogsopsworks/logrecipe.png %}

# Including application in Apache access logs

The other change we need to make is to include the application name in the Apache access logs, otherwise we won't
 be able to filter by application when creating a logging metric in CloudWatch.

To do this you need to override the Apache vhost template in the OpsWorks cookbooks. The recipe containing this template
 will depend on the application type, for example it's located in `mod_php5_apache2/templates/default/web_app.conf.erb` or
`passenger_apache2/templates/default/web_app.conf.erb` for PHP and Ruby applications respectively.

In this example we'll assume a PHP application, so create the following file in your custom cookbooks repository:

{% codeblock mod_php5_apache2/templates/default/web_app.conf.erb %}
<VirtualHost *:80>
  ServerName <%= @params[:server_name] %>
  <% if @params[:server_aliases] && !@params[:server_aliases].empty? -%>
  ServerAlias <% @params[:server_aliases].each do |a| %><%= "#{a}" %> <% end %>
  <% end -%>
  DocumentRoot <%= @params[:docroot] %>

  <Directory <%= @params[:docroot] %>>
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

  LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" \"<%= @params[:name] %>\"" combinedwithapp

  LogLevel <%= node[:apache][:log_level] %>
  ErrorLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-error.log
  CustomLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-access.log combinedwithapp
  CustomLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-ganglia.log ganglia

  FileETag none

  RewriteEngine On
  <% if node[:apache][:version] == '2.2' -%>
  Include <%= @params[:rewrite_config] %>*
  RewriteLog <%= node[:apache][:log_dir] %>/<%= @application_name %>-rewrite.log
  RewriteLogLevel 0
  <% else -%>
  IncludeOptional <%= @params[:rewrite_config] %>*
  <% end -%>

  <% @environment.each do |key, value| %>
  SetEnv "<%= key %>" "<%= value %>"
  <% end %>

  <% if @params[:mounted_at] -%>
  AliasMatch ^<%= @params[:mounted_at] %>/(.*)$ <%= @params[:docroot] %>$1
  <% end -%>

  <% if node[:apache][:version] == '2.2' -%>
  Include <%= @params[:local_config] %>*
  <% else -%>
  IncludeOptional <%= @params[:local_config] %>*
  <% end -%>
</VirtualHost>

<% if node[:deploy][@application_name][:ssl_support] -%>
<VirtualHost *:443>
  ServerName <%= @params[:server_name] %>
  <% if @params[:server_aliases] && !@params[:server_aliases].empty? -%>
  ServerAlias <% @params[:server_aliases].each do |a| %><%= "#{a}" %> <% end %>
  <% end -%>
  DocumentRoot <%= @params[:docroot] %>

  SSLEngine on
  SSLProxyEngine on
  SSLCertificateFile <%= node[:apache][:dir] %>/ssl/<%= @params[:server_name] %>.crt
  SSLCertificateKeyFile <%= node[:apache][:dir] %>/ssl/<%= @params[:server_name] %>.key
  <% if @params[:ssl_certificate_ca] -%>
  SSLCACertificateFile <%= node[:apache][:dir] %>/ssl/<%= @params[:server_name] %>.ca
  <% end -%>
  SetEnvIf User-Agent ".*MSIE.*" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0

  <Directory <%= @params[:docroot] %>>
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

  LogLevel <%= node[:apache][:log_level] %>
  ErrorLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-error.log
  CustomLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-ssl-access.log combinedwithapp
  CustomLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-ssl-ganglia.log ganglia

  FileETag none

  RewriteEngine On
  <% if node[:apache][:version] == '2.2' -%>
  Include <%= @params[:rewrite_config] %>-ssl*
  RewriteLog <%= node[:apache][:log_dir] %>/<%= @application_name %>-rewrite.log
  RewriteLogLevel 0
  <% else -%>
  IncludeOptional <%= @params[:rewrite_config] %>-ssl*
  <% end -%>

  <% @environment.each do |key, value| %>
  SetEnv "<%= key %>" "<%= value %>"
  <% end %>

  <% if @params[:mounted_at] -%>
  AliasMatch ^<%= @params[:mounted_at] %>/(.*)$ <%= @params[:docroot] %>$1
  <% end -%>

  <% if node[:apache][:version] == '2.2' -%>
  Include <%= @params[:local_config] %>-ssl*
  <% else -%>
  IncludeOptional <%= @params[:local_config] %>-ssl*
  <% end -%>
</VirtualHost>
<% end -%>
{% endcodeblock %}

This configuration is based on the [default template](https://github.com/aws/opsworks-cookbooks/blob/release-chef-11.10/mod_php5_apache2/templates/default/web_app.conf.erb) so
 it may be best to start with the latest template file on GitHub.

Note the following lines which are relevant:

`LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" \"<%= @params[:name] %>\"" combinedwithapp`

This is creating a new log format called `combinedweithapp`, it's the same as the `combined` format except the app name
 is appended to the end.

`CustomLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-access.log combinedwithapp`
`CustomLog <%= node[:apache][:log_dir] %>/<%= @params[:name] %>-ssl-access.log combinedwithapp`

This is telling both the HTTP and HTTPS access logs to use the new custom format.

# Testing the logging

If you now launch an instance, wait for it to come online then load the App a few times you should begin to see logs
 appearing in CloudWatch after a few minutes.

{% img /images/posts/cwlogsopsworks/logsappearing.png %}

This indicates that the CloudWatch agent is working, you should also be able to see the app name. You'll notice
 that logs are nicely categorized by stack and instance too.

# Creating a metric to monitor HTTP requests by App

How that we have Apache access logs being sent to CloudWatch including the App name we can setup a metric to monitor
 the number of requests made to this application.

From the "Log Groups" screen in CloudWatch, tick the appropriate log group (`cw-logs-test-http-access` in this example) and then
 click "Create Metric Filter" at the top.

Enter the following Filter Pattern:

`[host, logName, user, timestamp, request, statusCode, size, referer, useragent, app=app1, ...]`

Note that you should replace app1 with the name of the app you're interested in.

{% img /images/posts/cwlogsopsworks/metricfilterapp.png %}

You can test your filter pattern, otherwise proceed by clicking "Assign Metric".

{% img /images/posts/cwlogsopsworks/metric-name.png %}

Give your new metric an appropriate namespace (group) and name. As we're only interested in the number of requests the
 "Metric Value" is 1 (1 per request).

# Viewing the results

Once you've created the metric you won't see data until more logs matching that criteria occur, so either generate some
 traffic on your app or wait for some to come in. There can be a delay of about 5 minutes for metrics data to appear.

After data has been recorded into the metric you should be able to find that metric either by searching the metrics list
 or from the "Custom Metrics" drop down which will appear after you refresh the page.

{% img /images/posts/cwlogsopsworks/request_graph.png %}

Change the aggregation type from "average" to "sum" and you should now see a nice graph of the requests going to your app
 over time. You can change the interval to 1 minute to get the most detailed graph.

# Other metrics

In this example we're only interested in the number of HTTP requests, but you can easily create additional metrics for things
 like 4xx errors or response size (to get an estimate of bandwidth usage by app).

To graph 4xx errors use a Filter Pattern like this (note the statusCode field):

`[host, logName, user, timestamp, request, statusCode=4*, size, referer, useragent, app=app1, ...]`

To graph response size enter `$size` instead of 1 as the "Metric Value" when creating the metric.

To monitor multiple apps simply create additional sets of your custom metrics, each with a different app filter.



