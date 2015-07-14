---
layout: post
title: "Writing functions for AWS Lambda using NPM and Grunt"
date: 2014-12-07 11:51:16 +1000
comments: true
categories:
 - AWS
 - Lambda
 - NPM
 - Node.js
 - Grunt
---

AWS [recently announced](http://aws.amazon.com/blogs/aws/run-code-cloud/) a new compute product called [Lambda](http://aws.amazon.com/lambda/)
 allowing users to run Node.js functions on full managed infrastructure while paying only for the actual compute time
 used.

[NPM](https://www.npmjs.org/) is the primary package manager for Node.js, and while Lambda does not provide explicit
 NPM support it is possible to bundle NPM packages with your function to leverage 3rd party modules.

[Grunt](http://gruntjs.com/) is a task runner for JavaScript, allowing easy automation of project tasks such as building
 and packaging. Recently I [released a Grunt plugin](https://github.com/Tim-B/grunt-aws-lambda) to assist in testing Lambda
 functions and packaging functions including NPM dependencies.

This blog post provides an example of how to use NPM, Grunt and the grunt-aws-lambda plugin to create a Lambda function
 which will scrape a web page and save a list of links within that page to S3 using the cheerio NPM package.

<!-- more -->

# Before starting

It is assumed that you have Node.js (and NPM) installed on your system. Also, you should have `grunt-cli` installed
 globally.

This guide also assumes you have AWS credentials configured on your system. These will be used to both test the function
 and upload it to Lambda. The easiest way to install the [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) and run [aws configure](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html).
 Afterwards make sure ~/.aws/credentials is populated.

For more information on the required AWS IAM permissions [read here](https://github.com/Tim-B/grunt-aws-lambda#aws-permissions), you will also need whatever permissions are required to invoke your function (eg. access to S3 buckets).

# Creating your project

First, create a directory for your project and run `npm init`. After following the prompts you should end up with a
 package.json, open this file and edit values as necessary. Also add `grunt` and `grunt-aws-lambda` to the devDependencies.
 Don't forget to update the version numbers if new releases are made in the future.

Your package.json should looks something like the following.

{% codeblock lang:json package.json %}
{
  "name": "link-scraper",
  "version": "1.0.0",
  "description": "Scrapes a page and saves links in a HTML page to S3",
  "private": "true",
  "devDependencies": {
      "grunt": "0.4.*",
      "grunt-aws-lambda": "0.8.0"
  },
  "author": "",
  "license": "BSD"
}
{% endcodeblock %}

Next, create a Gruntfile.js with the following:

{% codeblock lang:javascript Gruntfile.js %}
var grunt = require('grunt');
grunt.loadNpmTasks('grunt-aws-lambda');

grunt.initConfig({
    lambda_invoke: {
        default: {
        }
    },
    lambda_deploy: {
        default: {
            arn: 'arn:aws:lambda:us-east-1:123456781234:function:my-function'
        }
    },
    lambda_package: {
        default: {
        }
    }
});

grunt.registerTask('deploy', ['lambda_package', 'lambda_deploy']);
{% endcodeblock %}

Then create an index.js, .npmignore and event.json with the following:
{% codeblock lang:javascript index.js %}
exports.handler = function (event, context) {
    console.log('webpage = ' + event.webpage);
    context.done(null, 'link-scraper complete.');
};
{% endcodeblock %}

{% codeblock .npmignore %}
event.json
Gruntfile.js
dist
*.iml
{% endcodeblock %}

{% codeblock lang:json event.json %}
{
    "webpage": "http://en.wikipedia.org/wiki/Main_Page"
}
{% endcodeblock %}

Now run `npm install`

Then run `grunt lambda_invoke`, you should receive the following output:

{% codeblock lambda_invoke %}
Running "lambda_invoke:default" (lambda_invoke) task

webpage = http://en.wikipedia.org/wiki/Main_Page

Message
-------
link-scraper complete.

Done, without errors.

{% endcodeblock %}

Congratulations, you've created a Lambda function and executed it locally!

# Using NPM packages with AWS Lambda

For most nontrivial functions you're going to want to leverage 3rd party libraries. The grunt plugin makes using NPM packages
 with Lambda easy.

In this example we're going to use the following NPM packages:

* [request](https://www.npmjs.org/package/request) - Make a HTTP request to download the target page
* [cheerio](https://www.npmjs.org/package/cheerio) - Query the DOM of the page we download
* [moment](https://www.npmjs.org/package/moment) - Format the current time
* [mustache](https://www.npmjs.org/package/mustache) - Generate our HTML page using a template file
* [aws-sdk](https://www.npmjs.org/package/aws-sdk) - Access the AWS API to upload the page to S3

As the aws-sdk is already available in the Lambda environment we're only going to include it in the devDependencies, the
 rest belong in the regular dependencies.

Update your package.json file with the following:

{% codeblock lang:json package.json %}
...
"dependencies": {
    "cheerio": "0.18.0",
    "request": "2.49.0",
    "mustache": "0.8.2",
    "moment": "2.8.4"
},
"devDependencies": {
    "grunt": "0.4.*",
    "grunt-aws-lambda": "0.8.0",
    "aws-sdk": "2.0.23"
},
"bundledDependencies": [
    "cheerio",
    "request",
    "mustache",
    "moment"
],
...
{% endcodeblock %}

**Note that you must include any packages which are to be included in the Lambda package within the bundledDependencies list**.

Now run `npm install` again to install these new packages.

# Developing a Lambda function using NPM packages

Now we can actually develop the Lambda function in index.js, below is an example of a Lambda function to download
 the page provided in the webpage attribute of the event, extract all the links, convert them to absolute URLs, then
 generate a list of these links from a mustache template and upload it to S3.

Update your index.js with the following, don't forget to replace `mybucket` with your actual destination bucket:

{% codeblock lang:javascript index.js %}
var cheerio = require('cheerio');
var request = require('request');
var url = require('url');
var fs = require('fs');
var mustache = require('mustache');
var AWS = require('aws-sdk');
var moment = require('moment');

exports.handler = function (event, context) {
    request(event.webpage, function (err, response, body) {
        if (err) console.log(err, err.stack); // an error occurred

        var $ = cheerio.load(body);
        var links = [];

        AWS.config.apiVersions = {
            s3: '2006-03-01',
            // other service API versions
        };

        var s3 = new AWS.S3();

        $("a").each(function () {
            var anchor = $(this);
            var href = anchor.attr("href");
            var text = anchor.text();

            if (typeof href !== 'undefined') {
                var abs = url.resolve(event.webpage, href);
                if (text == '') {
                    text = abs;
                }

                var new_item = {
                    text: text,
                    url: abs
                };

                if (links.indexOf(new_item) === -1) {
                    links.push(new_item);
                }
            }

        });
        fs.readFile('template.html', 'utf8', function (err, data) {
            if (err) console.log(err, err.stack); // an error occurred
            var view = {
                links: links,
                page: event.webpage,
                time: moment().format('MMMM Do YYYY, h:mm:ss a')
            }
            var output = mustache.render(data, view);
            var s3_params = {
                Bucket: 'mybucket',
                Key: 'links.html',
                ContentType: 'text/html',
                Body: output
            };
            s3.putObject(s3_params, function (err, data) {
                if (err) console.log(err, err.stack); // an error occurred
                context.done(null, 'link-scraper complete.');
            });

        });

    })
};
{% endcodeblock %}

Also create template.html which will be used to generate the page:

{% codeblock lang:html template.html %}
{% raw %}
<!doctype html>
<html lang="">
<head>
    <title>List of links on {{page}}</title>
</head>
<body>
<h1>List of links on {{page}}</h1>

<ul>
    {{#links}}
    <li><a href="{{{url}}}" target="_blank">{{text}}</a></li>
    {{/links}}
</ul>
<p>
    Generated {{time}}
</p>
</body>
</html>
{% endraw %}
{% endcodeblock %}

If we run `grunt lambda_invoke` again the task should output:

{% codeblock lambda_invoke %}
Running "lambda_invoke:default" (lambda_invoke) task


Message
-------
link-scraper complete.

Done, without errors.
{% endcodeblock %}

Then, if we look in our target bucket there should be a file called links.html, if you view it in your browser you should see something like:

{% img /images/posts/lambda_example/linkspage.png %}

# Deploying to Lambda

Before running the deploy task in grunt, go to the Lambda section of the AWS console and create a function which matches
 the name in the lambda_deploy section of your Gruntfile. In the example above the function ARN is `arn:aws:lambda:us-east-1:123456781234:function:my-function`.

When creating the function select the "Hello World" template, as the code will be overwritten when we deploy a zip.

{% img /images/posts/lambda_example/lambda-list.png %}


If you've added a deploy task to your Gruntfile as above you can now run `grunt deploy`, otherwise run both the lambda_package and lambda_deploy tasks with
 `grunt lambda_package lambda_deploy`.

After running that you should see something like:

{% codeblock lambda_deploy %}
Running "lambda_package:default" (lambda_package) task
link-scraper@1.0.0 ../../../../../../tmp/1417936030856.854/node_modules/link-scraper
Created package at dist/link-scraper_1-0-0_2014-11-7-17-7-10.zip

Running "lambda_deploy:default" (lambda_deploy) task
Uploading...
Package deployed.

Done, without errors.
{% endcodeblock %}

Now if you go to the AWS console you should be able to successfully invoke the uploaded task:

{% img /images/posts/lambda_example/test_console.png %}

After running you should see the date at the bottom of the generated links.html has been updated.

If for whatever reason you need to access the zip package which was uploaded you can find it under the dist directory of
 your project.

Congratulations, you've now successfully deployed a Lambda function using NPM packages and Grunt! In future you can
 invoke this function manually, via another application using the SDK, or you could modify it to respond to one of the supported
 Lambda events such as an S3 Put event.










