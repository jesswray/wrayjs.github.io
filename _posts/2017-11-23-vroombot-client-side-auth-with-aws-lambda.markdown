---
layout: post
title:  "vroombot: client side auth with aws lambda"
---

## The idea
I've been working on server-side OAuth a lot lately.  It's made me curious: how could I integrate authorization in a purely client side app ... like a React app deployed to Github Pages?

There's nowhere to hide secrets on a single-page app.  If I don't want a server, I need to go serverless.

I thought the next stage of my mileage widget might involve a dedicated Twitter account that I text to set off the action.  But [webhooks are still in restricted beta at Twitter][twitter-webhooks] and ain't no one got time for that.  

Instead I could use [Twilio][twilio], a programmatic SMS service, to hit an AWS Lambda function that parses the text and writes to my database (and if I'm feeling fancy, posts to Twitter)!  Then when the app loads, it can hit another Lambda function that fetches from the database.

All the services I played with today are free. Let's go:

### The first stage: Text -> Twilio -> API Gateway -> Lambda

#### Twilio with Express and ngrok

First, sign up for Twilio.  I followed the [quick start guide][node-quick-start-guide] and was texting to my phone right away.

To receive texts, we need an API endpoint.  So I started locally with a [basic Express server][super-basic-express-server] and [ngrok to expose the endpoint][ngrok-for-webhooks].  This is just to help me work; I could have jumped right into AWS Lambda but it's much easier to find what's wrong when I control everything and can change one thing at a time.  

Make a directory and `cd` into it.  You probably want to `npm init` because having a `package.json` will come in handy soon.  Then `npm install express`.  You are ready to serve some internets.

I pasted this into my index.js file:

{% highlight javascript %}
const express = require('express')
const app = express()
const port = 3000

app.get('/', (request, response) => {
  console.log("I received a text!")
})

app.listen(port, (err) => {
  console.log(`server is listening on ${port}`)
})
{% endhighlight %}

That's it, that's your whole server. Run:

{% highlight javascript %}
node index.js
ngrok http 3000
{% endhighlight %}

You're on the Internet!  

Copy the url it gives you into the Twitter console as a webhook and set its action to 'GET'. Now, text Twilio.  You should see "I received a text!" in your console.

Side note: you'll notice your ngrok URL changes every time you start and stop ngrok.  Which means you have to go paste the new URL into Twilio again.  If this gets annoying you can sign up for ngrok and get static URLs.

So, I have webhooks working locally.  But this doesn't do me a lot of good because I don't actually want to run my own server.

#### AWS Lambda

This is the part I was dreading. I looked briefly at Lambda last year and found it confusing -- I didn't understand how to include dependencies or use environment variables, or what the arguments being passed to my function were or where I could find documentation on them.  Thankfully lots of people have these questions so I was able to find REAMS of documentation.

It turns out there's not a lot of difference between you running your file with `node index.js` and Lambda running it.  As long as your directory contains all your dependencies, you're golden.  You need to export a handler function from your Node file, zip up your directory, and upload the .zip file to Lambda. It calls the file and handler function you specify and your code runs just like it does on your machine.

I achieved success in the following way:

Download, install and configure the AWS CLI.  I had to step back from the latest version of Python to get it to work.  

Create a test Lambda function in the AWS web console.  (You can do this with the CLI, but I went back and forth between the two.)

You should be able to see your Lambda functions in the console like so:

{% highlight bash %}
aws lambda list-functions
{% endhighlight %}

Get rid of your old `index.js` and write your function.  I've already figured out how to text through Twilio, so to test including npm modules in my Lambda function, I decided to stick with that.  You can set environment variables in the AWS web console, but I added `dotenv` (`npm install dotenv`) because I am not to be trusted with environment variables and will commit them to version control the first chance I get.

Now `index.js` looks like this:

{% highlight javascript %}
// Load the environment variables.
require('dotenv').config()

// Twilio Credentials
const accountSid = process.env.ACCOUNT_SID;
const authToken = process.env.AUTH_TOKEN;

// require the Twilio module and create a REST client
const client = require('twilio')(accountSid, authToken);

// Export the handler function that Lambda will look for.
exports.handler = (event, context, callback) => {
  client.messages.create({
    to: process.env.MY_PHONE_NUMBER,
    from: process.env.TWILIO_PHONE_NUMBER,
    body: "Hello from Lambda!",
  });

  const response = {
    "statusCode": 200,
    "body": "This is required",
  };

  callback(null, response);
};
{% endhighlight %}

Do:
- Zip the CONTENTS of your directory, not the directory.  
- Recursively zip (`zip -r`) so you get the contents of your directory's contents.
- Export your handler function and name your index file what your Lambda function is expecting.  Defaults are `index.js` and `handler`.

Push your zipfile to AWS Lambda (I had an existing function named `vroombot` I created in the web console). For me, this whole process looked like this:

{% highlight bash %}
[09:30:20] code
// ♥ cd vroombot_lambda/
[09:30:27] (master) vroombot_lambda
// ♥ zip -r vroombot-zipped-up .
[09:30:45] (master) vroombot_lambda
// ♥ aws lambda update-function-code --function-name vroombot --zip-file fileb://vroombot-zipped-up.zip --publish
{% endhighlight %}

Writing this out got old quick, so I added it as a script in my `package.json`:

{% highlight bash %}
"scripts": {
  "deploy": "rm vroombot-zipped-up.zip && zip -r vroombot-zipped-up node_modules .env index.js && aws lambda update-function-code --function-name vroombot --zip-file fileb://vroombot-zipped-up.zip --publish"
},
{% endhighlight %}

and I can run `npm run deploy` to deploy the updated function to Lambda.

#### The API Gateway

To call the function, I need an API endpoint.  So I configured an AWS API Gateway to proxy a web request to my lambda function.  Unfortunately I've lost the exact tutorial I used.

Once you've set up your action you can hit "Test" in the API Gateway console.  When I did, I got a text.

But to get this on the Internet, I need to [deploy the API][aws-api-gateway-deployment-docs].  I guessed that my `rest-api-id` was the number on the top of the screen and I was right.

{% highlight bash %}
[10:03:53] (master) vroombot_lambda
// ♥ aws apigateway create-deployment --region us-east-1 --rest-api-id MY_REST_API_ID --stage-name incoming
{% endhighlight %}

Mine returned what I presume is my deployment id. Go to your console, click Stages, and you should now see an Invoke URL. Paste it into your browser and you should get a text!

This URL is what I'm going to [turn around and give to Twilio][configure-twilio-incoming-phone-numbers] instead of the ngrok url.

And now when I text my Twilio number, it texts me back "Hello from Lambda!".

[aws-api-gateway-deployment-docs]: http://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-deployments.html
[twilio]: https://www.twilio.com/
[twitter-webhooks]: https://developer.twitter.com/en/docs/accounts-and-users/subscribe-account-activity/overview
[configure-twilio-incoming-phone-numbers]: https://www.twilio.com/console/phone-numbers/incoming
[ngrok]: https://ngrok.com/
[node-quick-start-guide]: https://www.twilio.com/blog/2016/09/how-to-send-an-sms-with-node-js-using-twilio.html
[ngrok-for-webhooks]: https://www.twilio.com/blog/2013/10/test-your-webhooks-locally-with-ngrok.html
[search-query-parser]: https://github.com/nepsilon/search-query-parser
[super-basic-express-server]: https://blog.risingstack.com/your-first-node-js-http-server/
