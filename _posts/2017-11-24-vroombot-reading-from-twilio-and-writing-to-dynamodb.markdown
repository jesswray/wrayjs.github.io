---
layout: post
title:  "vroombot: reading from twilio and writing to dynamodb"
---
### Part 2: Reading the response and writing to a database

#### Cloudwatch

Now all the services are talking.  But what are they saying?  To see what Twilio is passing to my function (hopefully my text message!) I put in some logging and went to Cloudwatch, where all Amazon Lambda functions automatically log.

The first argument looked like this:
{% highlight javascript %}
{
    "resource": "/",
    "path": "/",
    "httpMethod": "GET",
    "headers": {
      ...
    },
    "queryStringParameters": {
        ...
        "Body": "miles:8700 gallons:3.45 dollars:9.77",
        ...
    },
{% endhighlight %}

Now we parse it.  I used [`search-query-parser`][search-query-parser].

{% highlight javascript %}
const searchQuery = require('search-query-parser');
const queryOptions = { keywords: ['miles', 'dollars', 'gallons'] }

exports.handler = (event, context, callback) => {
  const body = selectn('queryStringParameters.Body', event) || {};
  const { miles, dollars, gallons } = searchQuery.parse(body, queryOptions);

  ...
{% endhighlight %}

#### DynamoDB

Following [this tutorial][dynamodb-tutorial], I went back to the console to figure out DynamoDB.

Since DynamoDB is NoSQL it doesn't have an auto-incrementing primary key. I have a natural primary key in my timestamp, so I used that.

DO
- Adjust the IAM permissions of the user that created your lambda function to include AmazonDynamoDBFullAccess.
- Provide key for new objects

I got it working like so:

{% highlight javascript %}
var AWS = require('aws-sdk');
AWS.config.update({ region: MY_REGION });
ddb = new AWS.DynamoDB({
  apiVersion: '2012-10-08'
});

var dbParams = {
  TableName: 'fillups',
  Item: {
    'gallons' : { N: gallons },
    'miles' : { N: miles },
    'dollars': { N: dollars },
    'time': { N: String(Date.now()) },
  },
};

ddb.putItem(dbParams, function(err, data) {
  if (err) {
    console.log("Error", err);
    textMe('Something went wrong.')
  } else {
    console.log("Success", data);
    textMe('It worked!')
  }
});

{% endhighlight %}

It's a wild non-relational world out there.

### It works!

I can text a database.

Next up, making my React app fetch from it!

[dynamodb-tutorial]: http://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/dynamodb-example-table-read-write.html https://www.twilio.com/blog/2013/10/test-your-webhooks-locally-with-ngrok.html
[search-query-parser]: https://github.com/nepsilon/search-query-parser
