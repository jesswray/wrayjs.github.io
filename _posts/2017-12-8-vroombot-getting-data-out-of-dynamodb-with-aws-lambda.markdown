---
layout: post
title:  "vroombot: getting data out of dynamodb with aws lambda"
---

## Part 3:
So I built a single-page React app on Github Pages.  And I have a database that I can text.  I want to get them talking to each other.

With the scary AWS setup out of the way, this should be fun!

I plan to create another AWS API Gateway endpoint.  I'll connect that to a new AWS Lambda function that fetches from my DynamoDB database and responds with JSON.  Finally I'll add an AJAX call to my React app that calls out to my endpoint on page load.

### Back to AWS

#### A new Lambda function

The [AWS docs on fetching from DynamoDB](dynamodb-scan) say you generally shouldn't use SCAN to fetch a whole table in a real app. However, we are building a bot that texts from motorcycles, so SCAN sounds perfect.  I followed this [Node.js DynamoDB code](dynamodb-scan).

{% highlight javascript %}
// Load the SDK for JavaScript
var AWS = require('aws-sdk');
// Set the region
AWS.config.update({
  region: 'us-east-1',
  endpoint: 'https://dynamodb.us-east-1.amazonaws.com'
});

const docClient = new AWS.DynamoDB.DocumentClient();
// No other params to specify -
// we want the whole table and every field in it.
const params = { TableName: "fillups" };
const onScan = (err, data) => {
  if (err) {
    console.log('error: ', err);
  } else {
    console.log('data: ', data);
  }
  return;
};

docClient.scan(params, onScan);
{% endhighlight %}

And it works!  I had one test item in the database.

{% highlight bash %}
// ♥ node index.js
data:  { Items: [ { dollars: 94, time: 1511493042794, miles: 8, gallons: 34 } ],
  Count: 1,
  ScannedCount: 1 }
{% endhighlight %}

#### Proxy Lambda from API Gateway

Just like we did before when setting up a Lambda function and an API gateway.

DO remember to call `JSON.stringify` on the body of your response.

Here's what my lambda function looks like now:

{% highlight javascript %}
var AWS = require('aws-sdk');
AWS.config.update({
  region: 'us-east-1',
  endpoint: 'https://dynamodb.us-east-1.amazonaws.com'
});

const docClient = new AWS.DynamoDB.DocumentClient();
const params = { TableName: "fillups" };

exports.handler = (event, context, callback) => {
  const onScan = (err, data) => {
    const response = {
      "statusCode": err ? 500 : 200,
      "body": JSON.stringify(err || data),
    };

    callback(null, response);
  };

  docClient.scan(params, onScan);
};
{% endhighlight %}

I can paste the URL into my browser and see the JSON there.  SO CLOSE.

[dynamodb-scan]: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SQLtoNoSQL.ReadData.Scan.html
[dynamodb-node]: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStarted.NodeJs.04.html
