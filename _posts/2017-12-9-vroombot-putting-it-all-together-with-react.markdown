---
layout: post
title:  "vroombot: putting it all together with react"
---

## Part 4
### The home stretch: getting the data to my app

#### AJAX from React

All I have left to do is connecting the React app to its data.  This is a super simple use case for AJAX from React: showing someone else's data when my page loads.

In my case, I need to do this fetch because my data and my app don't come from the same server.  My app is being rendered by Github Pages.  My data lives far away in a tiny DynamoDB database, accessed by an AWS Lambda function behind an AWS API Gateway.

But this data could be anything coming from anywhere and the idea would be the same.  You don't have it; you want to get it and then display it.

The idea is to wrap my presentational component(s) in a higher order component that calls for the data. Right now my app is like so:

{% highlight javascript %}
import React from 'react';

const foo = 'My static content';

const App = () => (
  <div>{foo}</div>
);

export default App;
{% endhighlight %}

And I need to extract an outer component that does the call, like so:

{% highlight javascript %}
import React, { Component } from 'react';

class App extends Component {
  state = { foo: null }

  componentDidMount () {
    foo = getTheData(); // make the AJAX call
    this.setState({ foo }); // save the data into state
  }

  render () {
    return (
      <View foo={this.state.foo} />
    );
  }
};

// View receives foo as a prop.
// It doesn't know and doesn't care where its data comes from.
const View = ({ foo }) => (
  <div>{foo}</div>
);

export default App;
{% endhighlight %}

The upside of getting my data this way: super flexible!  Downside, it could fail and there's visible lag while the AJAX call loads, so I'll probably add error handling and a loading indicator.  

The first draft of my app had to change a little to use DynamoDB.  It's my first time using a non-relational database and when I switched from sorting on a primary key to sorting on timestamps, I broke it.  

The reason is pretty funny and not at all about the database. I'm used to sorting iso8601-formatted dates on the front end, which are strings and can be sorted like strings.  But since my dates originate in Javascript I'm saving them as milliseconds past the epoch, which are numbers -- and calling `sort` on numbers coerces them to strings (because Javascript) and compares them as words.  You must pass an explicit comparison function.  Instead of `things.sort(item => item.foo)` do: `things.sort((a, b) => (a.foo - b.foo))`.

I also got a new motorcycle, because I didn't want to backport old data.  Just kidding, but I actually did.  Now I have a fuel indicator light.  It's basically a car!  Since I no longer have to set (and be laser focused on) the trip odometer to keep from running out of gas, it seems silly that I was ever recording the trip miles separately. So now I'm only tracking three data points and the math is simpler.

With the fetch and new sorting, [vroombot](vroombot) is up!  Populated with fake data at the moment because it's snowing :(

[vroombot]: https://jesswray.com/vroombot
