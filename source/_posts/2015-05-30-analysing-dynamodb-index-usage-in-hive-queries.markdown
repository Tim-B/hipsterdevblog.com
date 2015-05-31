---
layout: post
title: "Analysing DynamoDB index usage in Hive queries"
date: 2015-05-30 19:55:11 +1000
comments: true
categories: 
 - CloudWatch
 - EMR
 - Hive
 - DynamoDB
---

Elastic Map Reduce allows you to conveniently [run SQL-like queries against DynamoDB using Hive](http://docs.aws.amazon.com/ElasticMapReduce/latest/DeveloperGuide/EMRforDynamoDB.html).
This overcomes many of the limitations of the built-in DynamoDB query functionality and makes it significantly
more useful for storing raw analytical data.

While the abstraction provided by this handler is pretty good, it is still subject to the same underlying throughput
and indexing limitations faced when accessing data through the DynamoDB API directly. In particular, access efficiency
is extremely sensitive to the use of appropriate indexes - full table scans are both slow and expensive. 

The documentation provides [some guidance](http://docs.aws.amazon.com/ElasticMapReduce/latest/DeveloperGuide/EMR_Hive_Optimizing.html)
with regard to performance optimisation, however it does not go into how the handler maps a Hive query to
a DynamoDB scan or query, nor under what circumstances indexes will be used to avoid scanning the entire table.

In this blog post you'll find several Hive queries run against an example DynamoDB table, along with the resulting
DynamoDB request to observe which indexes are used.

<!-- more -->

# Setup

## Example Table

These queries were run against the following example table:

{% codeblock %}
+---------+-----------------+------+------+
| product | sequence_number | host | time |
+---------+-----------------+------+------+
{% endcodeblock %}

The following index structure was used:
{% codeblock %}
+------------------------+----------------------------------+
| Primary Index          | product, sequence_number (range) |
| Global Secondary Index | host, sequence_number (range)    |
| Local Secondary Index  | time                             |
+------------------------+----------------------------------+
{% endcodeblock %}

## Hive Table

{% codeblock lang:sql %}
CREATE EXTERNAL TABLE product_views ( 
    product bigint,
    sequence_number string,
    host string,
    time bigint
) STORED BY 'org.apache.hadoop.hive.dynamodb.DynamoDBStorageHandler' 
TBLPROPERTIES (
    "dynamodb.endpoint"="http://dynamodb.us-east-1.amazonaws.com",
    "dynamodb.table.name"="web-analytics-raw",
    "dynamodb.column.mapping" = "sequence_number:sequence_number,
    host:host,product:product,time:time"
);
{% endcodeblock %}

Note that the endpoint has been made the http (as oppose to https) to make the requests easier to observe without
encryption.

## Observing HTTP requests with wireshark

First, SSH into a core task node (it's easiest if you only run one) and run `sudo yum install tcpdump`, then from your
local machine:

{% codeblock lang:bash %}
$ mkfifo /tmp/wireshark
$ ssh -i ~/.ssh/keypair.pem hadoop@ec2-123.456.123.45.compute-1.amazonaws.com "sudo tcpdump -i eth0 -s 0 -U -w - not port 22" > /tmp/wireshark
{% endcodeblock %}

Finally, run the following in another terminal:
{% codeblock lang:bash %}
$ wireshark -k -i /tmp/wireshark
{% endcodeblock %}

# Queries

## Simple query on primary index key.

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 3;
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "KeyConditions":{
      "product":{
         "AttributeValueList":[
            {
               "N":"3"
            }
         ],
         "ComparisonOperator":"EQ"
      }
   },
   "ReturnConsumedCapacity":"TOTAL"
}
{% endcodeblock %}

Consumed capacity: 0.5

As you'd expect the handler is able to query for a specific key and save a full scan.

## Simple query on primary range key

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 3 
AND sequence_number > '49551126539595599111737467812411851738953944699026538498';
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "KeyConditions":{
      "sequence_number":{
         "AttributeValueList":[
            {
               "S":"49551126539595599111737467812411851738953944699026538498"
            }
         ],
         "ComparisonOperator":"GT"
      },
      "product":{
         "AttributeValueList":[
            {
               "N":"3"
            }
         ],
         "ComparisonOperator":"EQ"
      }
   },
   "ReturnConsumedCapacity":"TOTAL"
}
{% endcodeblock %}

Consumed capacity: 0.5

Likewise, a query is used for both the primary and range key.

## Query on primary key with limit

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 4 LIMIT 3;
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":1933,
   "KeyConditions":{
      "product":{
         "AttributeValueList":[
            {
               "N":"4"
            }
         ],
         "ComparisonOperator":"EQ"
      }
   },
   "ReturnConsumedCapacity":"TOTAL"
}
{% endcodeblock %}

Consumed capacity: 0.5

It appears however that limit conditions do not affect the DynamoDB query, and all results for the key
are returned regardless.

## Query on local index

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 3 AND time > 1432979370486;
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "KeyConditions":{
      "product":{
         "AttributeValueList":[
            {
               "N":"3"
            }
         ],
         "ComparisonOperator":"EQ"
      }
   },
   "ReturnConsumedCapacity":"TOTAL"
}
{% endcodeblock %}

Consumed capacity: 0.5

Unfortunately it doesn't seem local indexes are recognised, instead it just queries on the primary key and
filters the time from the returned results.

## Query on secondary global index

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE host = "site-8.com";
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

It seems secondary global indexes aren't supported either, leading to a full table scan in the absence of any
conditions on the primary key.

## Greater than query on primary index

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product > 5;
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

Unsurprisingly this results in a full scan given that you can only apply greater than conditions to a range key. 

## Using OR to select multiple primary keys

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 5 OR product = 6;
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

This results in a full scan also, which isn't much of a surprise seeing as you can only query on one key at a time.   

## Using multiple range key values with OR

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 6 AND 
(sequence_number = "49551126539595599111737467812353823299612434595900817410" OR 
sequence_number = "49551126539595599111737467812437239181165855278949728258");
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "KeyConditions":{
      "product":{
         "AttributeValueList":[
            {
               "N":"6"
            }
         ],
         "ComparisonOperator":"EQ"
      }
   },
   "ReturnConsumedCapacity":"TOTAL"
}
{% endcodeblock %}

Consumed capacity: 0.5

It starts by simply querying on the primary key then filtering the results, nothing fancy like selecting only those
greater than the smallest to reduce the returned results.

## Querying on range key alone

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views 
WHERE sequence_number > '49551126539595599111737467812411851738953944699026538498';
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

Full scan, which is expected as you can't query on a range key without specifying a primary key.

## Like query on range key

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 3 
AND sequence_number LIKE '49551126539595599111737467812411851738953944699026%';
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":2147483647,
   "KeyConditions":{
      "product":{
         "AttributeValueList":[
            {
               "N":"3"
            }
         ],
         "ComparisonOperator":"EQ"
      }
   },
   "ReturnConsumedCapacity":"TOTAL"
}
{% endcodeblock %}

Consumed capacity: 0.5

Although there is support for BEGINS_WITH in the KeyConditions field it appears this isn't used. Instead it just queries
on the primary key and filters the results for the range key condition.

## Primary key ID expressed indirectly through multiple conditions

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product > 2 AND product < 4;
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":1933,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

No luck on this one either, it doesn't recognise that the only key value can be 3 - although I'm not too surprised about that.
Be sure that your key values are explicit enough for the handler to recognise.

## Primary key using IN condition

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product IN (3);
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":1933,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

It will not recognise a single ID in an IN statement either, resulting in a full scan.

## Subquery on primary key

Hive Query:
{% codeblock lang:sql %}
SELECT * FROM product_views AS p1 WHERE p1.product IN (SELECT p2.product FROM product_views AS p2 WHERE p2.product = 4);
{% endcodeblock %}

DynamoDB Request:
{% codeblock lang:json %}
{
   "TableName":"web-analytics-raw",
   "Limit":1933,
   "ReturnConsumedCapacity":"TOTAL",
   "TotalSegments":1,
   "Segment":0
}
{% endcodeblock %}

Consumed capacity: 2.5

Surprisingly only one scan request was made as far as I could tell, so while unfortunately it wasn't able to issue
   this as two queries at least it didn't issue two scans.

# Conclusion

Unfortunately it seems that neither global secondary indexes or local indexes are supported, however scenarios involving
a query on a single primary key are recognised pretty well. This makes it practical to use a primary key as a method
of partitioning your data to avoid EMR queries taking longer over time as the table grows.

With that in mind you may also be able to design queries which avoid a full scan but still achieve the same outcome.

For example:
{% codeblock lang:sql %}
SELECT * FROM product_views WHERE product = 5 OR product = 6;
{% endcodeblock %}

Could be achieved with:
{% codeblock lang:sql %}
DROP TABLE IF EXISTS tmptable;

CREATE TABLE tmptable ( 
  product bigint,
  sequence_number string,
  host string,
  time bigint
);
 
INSERT INTO TABLE tmptable SELECT * FROM product_views WHERE product = 4;
INSERT INTO TABLE tmptable SELECT * FROM product_views WHERE product = 5;
 
SELECT * FROM tmptable; 
{% endcodeblock %}

Which would require two query requests instead of one scan.

It's also worth noting that filtering on the DynamoDB side is never used, although that isn't a huge issue seeing
  as throughput consumption is calculated before any filters are applied.

You can also use the EXPLAIN Hive command to give you some clue about how a query will be executed, however as far as
I'm aware the raw DynamoDB request isn't exposed in either the EXPLAIN output or the query logs. I'd be glad to know if there
is a more convenient way to view the raw DynamoDB request than inspecting the network traffic.










