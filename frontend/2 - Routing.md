Agent 5 Frontend - Routing
==========================

Routing is very important for Single Page Applications (SPA for short). It
allows the user and the browser to keep track of the state of the application,
and to be able to return to that state easily.

## Browser Location

The browser location refers to the current url that the browser is pointing to.

> http://google.com/

vs

> http://google.com/results

In normal websites, this value is automatically updated when you click on a link
because the browser is navigating to a new url and doing a GET request at that
url. However in SPAs, the browser isn't navigating to a new url, but is usually
doing an XHR behind the scenes to get new data. This means that the browser
doesn't automatically update the location when a link is clicked.

### Why is this important?

Updating the location allows for a design architecture called REST, or
Representational State Transfer. Basically, since a server has no knowledge of
the "state" a user is in (for example, what page the user is viewing), REST uses
the browser location to decode that state.
Let's see an example of this:

Suppose I visit my new SPA, `http://bestsiteever.com`. On that site, I have an
"About" page. If I click a link to show the "About" page, but don't update the
location of the browser, the browser will still point to
`http://bestsiteever.com`. Now suppose I want to bookmark this "About" page.
Since the browser location hasn't been updated, I am effectively bookmarking the
"Home" page. However, if I update the location manually to be
`http://bestsiteever.com/about` I can bookmark the "About" page directly.

## Routing in Backbone

### Location
We can update the location of the browser by using Backbone's built-in `history`
functionality and Marionette's [AppRouter](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.approuter.md).

The AppRouter allows us to define new routes in our app, and what should happen
when those routes are visited. Here's an example:

```js
// Create an ItemView named 'MyView'
...
// Create our usual Controller with a 'show' method
var MyController = Marionette.Controller.extend({
  show: function(region) {
    var myView = new MyView();

    region.show(myView);
  }
});

var MyRouter = Marionette.AppRouter.extend({
  // Define the new routes the app should have. In this case, we are adding an
  // 'about' route so that when Backbone navigates to
  // http://bestsiteever.com/about, it will fire the associated function, 'show'
  appRoutes: {
    'about': 'show'
  }
});

var myController = new MyController();

// When creating our router, we must pass in a controller object that contains
// all the methods the router can call. Our controller has a 'show' method, so
// it is a valid object to pass to the router
var myRouter = new MyRouter({ controller: myController });
```

And that is all you need to create a new route for your app. When the user
clicks on a link to view the "About" page, the browser location will be updated
and a new view will be rendered.

### Navigation

Navigation refers to how the user gets to a page versus what happens when a
user goes to a page. Going back to a normal website, you simply create an
hyperlink with an `href` attribute:

```html
<a href="about">About</a>
```

This won't work by itself for SPAs because the default behavior of the browser
will be to navigate to the new URL, do a GET and render the response. We want
Backbone to handle the routing, not the browser.

There are two ways to handle this. One is to use `Backbone.history.navigate`
directly which will cause Backbone to route to the desired location. Another is
to override the functionality of clicking on an hyperlink, parse out the
location the user is trying to go and then use `Backbone.history.navigate` to go
there. We use both techniques in Agent 5.

```js
var MyView = Marionette.ItemView.extend({
  template: '<div>Hello World</div>',

  // The events hash contains events to listen for and the action to initiate
  // when that event happens
  events: {
    // Listen for the 'click' event on any element that has the class
    // 'about-link'. When that event happens, call the 'goToAbout' method
    'click .about-link': 'goToAbout'
  },

  // Define goToAbout that will be called when '.about-link' is clicked.
  // The 'e' parameter is the event object passed in from the event
  goToAbout: function(e) {
    // First, we want to block the browser from doing its default behavior
    e.preventDefault();

    // Now manually navigate to where we should go. The first parameter is the
    // location to go to, the second determines whether Backbone should also
    // trigger the associated action with that route. It should almost always
    // be 'true'
    Backbone.history.navigate('about', true);
  }
});
```

The other way is to add an attribute to the hyperlink that serves to flag this
link as a SPA link:

```html
<a data-link="spa" href="about">About</a>
```

Agent 5 already has code that will catch clicks on links and, if they have the
`data-link="spa"` attribute will use Backbone to route instead of the browser.