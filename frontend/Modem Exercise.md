Agent 5 Frontend - Modem Exercise
=================================

In this exercise, we will build the frontend for creating and showing a Modem
object. This will cover the following topics:

- Routing on the frontend
- Creating a form
- Using Handlebars helpers
- Submitting data to the server
- Fetching data from the server
- Rendering templates

## Creating a Modem

### The Template

When creating a new page, the easiest place to start is the template. This helps
you figure out what data you need and what interactions will be available to the
user. Once these requirements are established, then you can create a module that
enables that functionality. Let's see what a create modem template would look
like:

```html
<div class="container">
  <div class="row">
    <div class="span6">
      <h2>Create Modem</h2>

      <form id="create-modem-form">
        {{ editFor "deviceName" }}
        {{ selectFor "deviceType" listName="MODEM_TYPE" }}
        {{ selectFor "hostname" listName="HOSTNAME" }}
        <button class="btn btn-primary btn-create-modem">Save</button>
      </form>
    </div>
  </div>
</div>
```

Here we are introduced to a Handlebars concept called "Helpers". Helpers allow
for executing custom code inside a template. This custom code can determine
what to render depending on certain values given. For example,
`{{ editFor "deviceName" }}` will create a simple `label` and `input` for the
property "deviceName".

```html
<div class="control-group">
  <label for="deviceName" class="control-label">Device Name</label>
  <div class="controls">
    <input id="deviceName" class="input-block-level" type="text" name="DeviceName" value>
  </div>
</div>
```

`{{ selectFor "deviceType" listName="MODEM_TYPE" }}` will
create a `select` for "deviceType" that will be populated by the entries
in the list "MODEM_TYPE". The helper files contain documentation on what options
they take and how to use them.

### Require.js Setup

```js
define([
  'app',
  'jquery',
  'underscore',
  'backbone',
  'marionette',
  'app/core/baseHelper',
  'hbs!templates/createModemTpl'
], function(app, $, _, Backbone, Marionette, BaseHelper, createModemTpl) {
  'use strict';
```

### Model

Our model for this is going to be simple. We only have three properties and none
of them need any default values since the user will fill the values in. The one
thing we need to handle is which url to POST to when we save our model:

```js
  var Modem = Backbone.Model.extend({
    // The url property can either be a string or a function. We use a function
    // here because when the model is first defined, 'app.root' is not set. Only
    // during runtime is this variable set
    url: function() { return app.root + '/modems/create'; }
  });
```

### View

Next, let's build our Marionette view that will be responsible for rendering the
template and handling any user interaction (such as clicking the save button):

```js
  var ModemView = BaseHelper.ItemView.extend({
    template: createModemTpl,

    // BaseHelper features
    hasLists: true,

    events: {
      'click .btn-create-modem': 'submit'
    },

    submit: function(e) {
      e.preventDefault();

      // Save our data
      this.model.save({
        DeviceName: this.$('#deviceName').val(),
        DeviceType: this.$('#deviceType').val(),
        HostName: this.$('#hostname').val()
      }, {
        success: function(model, res) {
          // Route the user to the show page
          Backbone.history.navigate('modems/' + res.target.id, true);
        }
      });
    }
  });
```

One thing that should be pointed out is that instead of extending the basic
`Marionette.ItemView`, we are instead extending `BaseHelper.ItemView`. The
BaseHelper is a custom Agent 5 component that contains many useful features that
can be opted in to. In this case, we want to use its list population feature
which we enable with the `hasLists: true` property.

### Controller

Our controller will be a simple one that contains a basic `show` method:

```js
  var Controller = Marionette.Controller.extend({
    show: function(region) {
      var modem = new Modem();
      var modemView = new ModemView({ model: modem });

      region.show(modemView);

      return modem;
    }
  });
```

### Returning the Module Object

Now that we have all our components built, we need to construct our controller
and return it so that Require.js has an object it can inject into other modules:

```js
  var controller = new Controller();
  return controller;
});
```

## Showing a Modem

Now that we've created a modem, we need to display this information to the user.
To do this, we'll use a couple of new techniques so that we can get a better
handle on how to use them. This includes using a Marionette [Layout](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.layout.md)
and a Marionette [AppRouter](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.approuter.md).

### The Templates

To show the new modem information to user in a clear manner, we're going to
split up its data between different templates. This will make it easier to
move certain details to a different part of the page if you want the layout to
change. Keep in mind this is a simple object so these two templates could be
combined, but this is also a great way to get introduced to the power of
regions.

First, let's make our Header template. This will contain the Modem name and the
Host name of the Modem:

```html
<div class="pull-right">
  {{> contextMenuTpl }}
</div>

<h3 class="itemsHeader">Modem: {{ deviceName }} installed on {{ hostName }}</h3>
```

This template introduces another powerful feature of Handlebars, partials. A
partial is just a snippet of html that you want to be able to use in multiple
different places throughout the app. In this case, there is a context menu that
we use often in Agent 5 on different pages, which made it a great candidate to
be a partial.

Now let's make our Details template. This will contain any other details about
the modem, such as the device type and status:

```html
<table class="table table-condensed table-striped table-border modem-details">
  <tbody>
    <tr>
      <td class="md"><strong>Device Type</strong></td>
      <td class="state">{{ deviceType }}</td>
    </tr>

    <tr>
      <td class="md"><strong>STATUS</strong></td>
      <td class="status">{{ status }}</td>
    </tr>
  </tbody>
</table>
```

Lastly, let's create a parent template that will give us DOM elements that we
can use to put these Header and Details templates inside of:

```html
<div id="show-modem" class="content">

  <div class="header" />

  <div class="row-fluid">
    <div class="details span4 padded-wrapper" />
  </div>
</div>
```

This is just an empty template as far as content goes, but it gives us some
elements that we can attach regions to later on, specifically '.header' and
'.details'.

### Require.js Setup

```js
define([
  'app',
  'jquery',
  'underscore',
  'backbone',
  'marionette',
  'app/core/baseHelper',
  'hbs!templates/modem/modemLayoutTpl',
  'hbs!templates/modem/modemDetailsTpl',
  'hbs!templates/modem/modemHeaderTpl'
],
function (app, $, _, Backbone, Marionette, BaseHelper, modemLayoutTpl, modemDetailsTpl, modemHeaderTpl) {
  'use strict';
```

### Model

Create a simple model that has the url we need to query the backend for the
Modem data:

```js
  var Modem = Backbone.Model.extend({
    url: function() { return app.root + 'modems/' + this.get('id'); }
  });
```

### Layout

A Layout is simply and ItemView with functionality added to create Regions. The
way you create these regions is by passing a `regions` hash to the Layout where
the key is going to be the attribute name and the value is the DOM selector for
the element in the Layout's template the region should bind to:

```js
  var ModemLayout = Marionette.Layout.extend({
    template: modemLayoutTpl,

    regions: {
      header: '.header',
      details: '.details'
    }
  });
```

Layout regions cannot be accessed until after the Layout has been rendered, so
do not try to manipulate those regions before then.

### Other Views

```js
  var HeaderView = Marionette.ItemView.extend({
    className: 'page-header',
    template: modemHeaderTpl
  });

  var DetailsView = Marionette.ItemView.extend({
    template: modemDetailsTpl
  });
```

### Controller

```js
  var Controller = Marionette.Controller.extend({
    show: function(region, id) {
      var self = this;

      var modem = new Modem({ id: id });
      modem.fetch({
        success: function() {
          // Global event that changes the title of the document
          app.vent.trigger('change:document:title', modem.get('deviceName'));

          // Create and render the layout
          self.layout = new ModemLayout();
          region.show(self.layout);

          // We now have access to that layout's regions, so we render our other
          // views inside those regions
          var headerView = new HeaderView({ model: modem });
          self.layout.header.show(headerView);

          var detailsView = new DetailsView({ model: modem });
          self.layout.details.show(detailsView);
        }
      });
    }
  });

  var controller = new Controller();
  return controller;
});
```

## Routing

The last thing we need to handle is the frontend routing. Backbone has a built-
in routing mechanism that we can leverage to add our own routes. Frontend
routing allows for seemless transitions between views without having the entire
page flash and reload like normal routing. In order to create new routes for our
modem feature, we're going to create separate module that is specifically for
handling routing. Let's jump into what that looks like:

```js
define([
  'app',
  'jquery',
  'underscore',
  'marionette',
  'app/custom/createModem',
  // 'app/custom/editModem',
  'app/custom/showModem'
],
function(app, $, _, Marionette, createModem, /*editModem,*/ showModem) {
  'use strict';

  var ModemRouter = Marionette.AppRouter.extend({
    appRoutes: {
      'modems/create': 'showCreate',
      'modems/edit': 'showEdit',
      'modems/:id': 'showModem'
    }
  });
```

When defining a router, you need to give it the routes it is responsible for. In
this case, it is responsible for 3 routes: `modems/create`, `modems/edit`, and
`modems/:id`. There are a couple of things to note here.

First, the last route, `modems/:id` is different from the other two because of
the `:` in its path. The `:` means "treat this part of the path as a variable".
That means `modems/123` and `modems/something` would both be handled by the
`modems/:id` route.

The second note is that the `modems/create` and `modems/edit` routes are defined
before `modems/:id`. That's because if they were defined after, the `modems/:id`
route would actually catch a navigation to 'modems/create' and treat 'create' as
the `:id`.

When instantiating the router, you need to give it a `controller`. The only
requirement for this controller is that it has the functions that the router
expects to be able to call. That means it doesn't necessarily have to be a
Marionette Controller, but can be any object that contains the necessary
functions. Let's build a routerController:

```js
  var routerController = {
    showCreate: function() {
      createModem.show(app.mainRegion);
    },

    showEdit: function() {
      // Stub
    },

    showModem: function(id) {
      showModem.show(app.mainRegion, id);
    }
  };

  var modemRouter = new ModemRouter({ controller: routerController });
  return modemRouter;
});
```

As you can see, the `routerController` is just a simple object that contains the
functions that our Router expects. This architecture makes it easy to
consolidate your routes in to one location.

Note, the `app.mainRegion` is the main region of the entire application set up
in app.js.

## Homework

Now that we've set up being able to create and show a Modem, let's build a
module that is responsible for editing a modem. Here are a couple of things to
keep in consideration as you solve this:

1. I should be able to navigate from the showModem page to the editModem page
2. When presented with the edit modem form, all the fields should be filled in
  with the existing data of the modem. Hint: read the editFor and selectFor
  documentation for help on setting default values.
3. I should be able to click a button to save my changes and be automatically
  navigated back to the showModem page
4. I should be able to cancel my changes and be automatically navigated back to
  the showModem page
5. If I refresh the edit form, nothing should change and I should be presented
  with the same form and data as if I navigated there from the showModem page.
  Keep in mind that a refresh will completely reload the app and all your modem
  data will be lost. Think of a way to get that data back once the app has
  reloaded.