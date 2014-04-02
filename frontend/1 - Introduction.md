Agent 5 Frontend - Introduction
===============================

## Tools
- Backbone with Marionette
- jQuery
- Require.js
- Handlebars (templating engine)
- Select2
- Moment.js
- Bootstrap
- Many other smaller plugins (view the `bower.json` file for a complete list)

## Architecture

### Modules

We use the MVC architecture to drive our modules

#### Model (The Data)
- The data that drives a View
- One-to-one relationship with a View
- Should not contain knowledge of a View
    - Meaning no use of jquery in the Model

#### View (The UI)
- Takes a model's data and builds a UI around it
- Handles user interaction with the UI

#### Controller (The Public API)
- For our application, the Controller serves as the public API for the module
- Almost always contains a `show` method
    - The first parameter should be a [Marionette Region](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.region.md)
      object where the module should be rendered
    - Any required information for the module should extra parameters after the
      region
    - Any optional information should be contained in an `options` hash that is
      the last parameter of the `show` method
    - Should return the model of the module. This can be leveraged for
      communication to the caller

#### An Example

First, a list of dependencies for the module for Require.js

```javascript
define([ // Require.js modules start with a define() call
  'app',
  'jquery',
  'underscore',
  'marionette',
  'hbs!templates/error/errorPageTpl'
],
```

Next, the function declaration

```javascript
function(app, $, _, Marionette, errorPageTpl) {
  'use strict'; // This enforces certain rules so that common mistakes aren't made
```

Now define our Model. Notice that all we need is to set some defaults for
this one. Models can contain more complicated code, but should only handle data
computation, and should never be aware of the DOM or View that owns it.

```javascript
  var Error = Backbone.Model.extend({
    defaults: {
      consoleUrl: app.base + 'console'
    }
  });
```

Then define our View. This is a very simple view that only uses a template.
Views can get much more complicated in order to handle user interactions with
the UI.

```javascript
  var ErrorView = Marionette.ItemView.extend({
    className: 'container',
    template: errorPageTpl // Notice this is a dependency that we've pulled in
  });
```

Let's take a break from Javascript for a second and look at our Handlebars
template. This is a very simple template, but shows the basics of using
Handlebars.

```html
<div class="alert alert-error error-page">
  <h3>{{ domain }} {{ id }} does not exist</h3>

  <p><a href="{{ consoleUrl }}">Return to the console</a></p>
</div>
```

See that we simply surround the Model attributes with `{{ attribute }}` when we
want to render that value.


Next, define our Controller. Consider this the API for this module that other
modules will use. We've defined our `show` method that takes the `region` first,
but then also requires a `domain` and `id`. Since there are
no optional parameters for this module, there is no `options` hash parameter.

```javascript
  var Controller = Marionette.Controller.extend({
    show: function(region, domain, id) {
      // Construct our model with the given data
      var error = new Error({
        domain: domain,
        id: id
      });

      // Create our view and pass the model to it
      var errorView = new ErrorView({ model: error });

      // Render the view in the desired region
      region.show(errorView);

      // Return the model so that the caller can listen in on model events if
      // desired
      return error;
    }
  });
```

Lastly, we should return an object that Require.js can inject into other modules
that depend on this one. We create a new Controller and return that. This is
called a singleton pattern because the app now has only one instance of the
module regardless of how many other modules depend on it.

```javascript
  var controller = new Controller();
  return controller;
});
```

Now that we have a module, let's use it:

```javascript
define([
  app/core/errorPage // This is our module
],
function(errorPage) { // This parameter is now the Controller instance we made in our module
  'use strict';

  // This region points to where we want the error page to be shown
  var region = new Marionette.Region({ el: 'body' });

  // Now let's call the 'show' method on our Controller instance, passing in all
  // the information it needs
  errorPage.show(region, 'My Domain', 123);
});
```

The resulting errorPage would render as:

```html
<div class="alert alert-error error-page">
  <h3>My Domain 123 does not exist</h3>

  <p><a href="/console">Return to the console</a></p>
</div>
```

#### Regions

Regions are the driving force of the UI layout. They are aptly named because
they represent areas in the layout that a module should be rendered. Regions
make module re-use very easy because simply passing in a different region to
the module allows it to render in a different part of the app. Let's look back
at our previous example to get a better understanding of this:

```javascript
var region = new Marionette.Region({ el: 'body' });
```

The `el` value in the hash indicates what element in the DOM this region should
be associated with, in this case the `body` of the page. Then we passed that
region object down to the errorPage module. Inside the `show` method, we had

```javascript
// Create model and view
...
region.show(errorView);
...
```

Here the errorPage module is telling the given region to render it's created
view. What if we wanted to render this error page in a different DOM element?

```javascript
var region = new Marionette.Region({ el: '.my-other-error-page' });

errorPage.show(region, 'My Different Domain', 456});
```

Now the errorPage's view will be rendered inside the `.my-other-error-page` DOM
element.

#### Layouts

Layouts are simply [Marionette ItemViews](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.itemview.md)
with Regions mixed in. This makes for easily creating Regions associated with
certain DOM elements in your Layout's template. Let's take a look:

```javascript
var MyLayout = Marionette.Layout.extend({
  // A basic template with two empty divs. I want to render child modules inside
  // these divs. The '_.template' is simply a helper function
  template: _.template(
              '<div class="first"></div>' +
              '<div class="second"></div>'
            ),

  // Create regions in the Layout that are associated with DOM elements in the
  // template
  regions: {
    firstRegion: '.first',
    secondRegion: '.second'
  }
});

var Controller = Marionette.Controller.extend({
  show: function(region) {
    // Create the Layout instance
    var myLayout = new MyLayout();

    // First, render the Layout so that the template exists in the DOM and the
    // layout regions are created
    region.show(myLayout);

    // We can now use our layout's regions to pass to other modules
    otherModule1.show(myLayout.firstRegion);
    otherModule2.show(myLayout.secondRegion);
  }
});
```
