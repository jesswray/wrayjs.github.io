---
layout: post
title:  "playing with create-react-app and github pages"
---

I made a mileage tracking widget with `create-react-app` for my bike.  

`create-react-app` is amazing -- it took me more time to read [this tutorial][deploying-react] than to get it deployed to Github Pages.

Previously I'd been keeping photos of my odometer and gas receipts on my phone.  For the purpose of this exercise I transcribed them into a file, like so:

{% highlight javascript %}
<!-- store.js -->

const store = [
  {
    id: 1,
    date: {
      month: 8,
      day: 21,
      year: 2017,
    },
    gallons: 2.8476,
    ppg: 2.529,
    price: 7.20,
    trip: 204,
    odometer: 5557,
  },
  ...
{% endhighlight %}

I know, it looks like it wants to be a database, but a database would be overkill for my purposes right now.  

In `App.js`, all I have to do is `import store from './store'` and we're in business.  I know, it's like it wants to be Redux, but ... also overkill.

I made two additions for creature comfort: `recompose` and `react-bootstrap`.  Recompose is a helper library that expresses common React idioms as higher order components, or as functions that return HOCs.  Bootstrap is so overused as to be totally uncool but whatever.  You don't have to use either, but I like them.

To start, I wanted to build a menu of entries that I can switch between.  Besides viewing
each entry, I also wanted a running "totals" section to view my average gas mileage.

{% highlight javascript %}
// App.js
import React from 'react';
import { compose, withProps, withStateHandlers } from 'recompose';
import './App.css';
import store from './store';
import DayLink from './components/DayLink';
import DayView from './components/DayView';
import Averages from './components/Averages';

const App = ({ store, onClick, selectedDay }) => (
  <div className="App">
    <div className="App-header lead">
      react mileage tracking
    </div>
    <div className='container'>
      <div className='row'>
        <div className='col-sm-2'>
          <h4>Dates</h4>
          {store.map(entry => (
            <DayLink
              entry={entry}
              key={entry.id}
              onClick={onClick}
              active={selectedDay === entry.id}
            />
          ))}
        </div>
        <DayView
          entry={store.find(entry => entry.id === selectedDay)}
          className='col-sm-6'
        />
        <Averages
          store={store}
          className='col-sm-6'
        />
      </div>
    </div>
  </div>
);

export default compose(
  withProps({ store }),
  withStateHandlers(() => ({
    selectedDay: 1,
  }), {
    onClick: () => id => ({ selectedDay: id }),
  }),
)(App);
{% endhighlight %}


{% highlight javascript %}
// DayView.js
import React from 'react';

const labels = {
  trip: 'Miles',
  gallons: 'Gallons',
  ppg: 'PPG',
  price: 'Price',
  odometer: 'Odometer',
};

const DayView = ({ entry, className }) => (
  <div className={className}>
    <h4>On this day ... </h4>
    {Object.keys(labels).map(key => (
      <div className={entry[key] ? 'text' : 'text-danger'}>
        {labels[key]}: {entry[key]}
      </div>
    ))}
  </div>
);

export default DayView;
{% endhighlight %}

{% highlight javascript %}
// DayLink.js
import React from 'react';
import { Button } from 'react-bootstrap';
import { withHandlers } from 'recompose';

const DayLink = ({ entry, handleClick, key, active }) => {
  const { month, day, year } = entry.date;
  return (
    <Button
      className='DayLink-button'
      key={key}
      onClick={handleClick}
      active={active}
    >
      {month}-{day}-{year}
    </Button>
  );
};

export default withHandlers({
  handleClick: ({ onClick, entry }) => () => onClick(entry.id),
})(DayLink);
{% endhighlight %}

The selected button gets an "active" state, categories with no data appear in red, and missing data is excluded from my averages.

View it at [react mileage tracking](react-mileage-tracking-1).

I'm disappointed to find out that I'm getting 48 miles to the gallon.  It could be as high as 65 but I spend too much time sitting in traffic.

[deploying-react]: https://medium.freecodecamp.org/surge-vs-github-pages-deploying-a-create-react-app-project-c0ecbf317089
[react-mileage-tracking-1]: https://jesswray.com/react-mileage-tracking-1
