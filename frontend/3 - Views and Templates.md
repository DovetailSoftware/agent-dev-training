Agent 5 Frontend - Views & Templates
====================================

Views drive the visual part of the app, the UI. There are multiple types of
views available through Marionette, including the [ItemView](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.itemview.md),
the [CollectionView](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.collectionview.md),
the [CompositeView](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.compositeview.md),
and the [Layout](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.layout.md).

To get more experience with this, let's add a new feature called "Modem" to
Agent 5. For our Modem feature, we will use Marionette's ItemView to drive
rendering our template.

Agent 5 uses [Handlebars](http://handlebarsjs.com/) as its templating engine.
Handlebars, like most templating engines, is based on HTML and adds
functionality on top of it. Here's an example of a Handlebars template:

```html
<div class="{{ customClass }}"> Hello {{ name }}</div>

{{#if age}}
<p>You are {{ age }} years old</p>
{{/if}}
```

Remembering from our Introduction, variables are surrouned by curly brackets.
Also, there's another Handlebars feature exampled, the helper
`{{if <variable>}}`. Helpers allow custom functionality to be run while the
template is compiling. In this case, we're simply checking if `age` exists and
if so, rendering a message that includes the age.

Helpers can add elements to the DOM, determine which element to render and many
other things. We will use multiple custom helpers as we build out our Modem
views.

## New Model Form

The first step in adding our Modem feature is enabling the creation of one. In
order to do this, a new Modem view and new Modem template need to be created.
Let's tackle the template first:

### Template

```html
<div class="new-modem">
  <form id="new-modem-form">
    {{ editFor "deviceName" }}
    {{ selectFor "deviceType" listName="MODEM_TYPE" }}
    {{ selectFor "hostname" listName="HOSTNAME" }}
    <button id="btn-save-modem">Save</button>
  </form>
</div>
```

We've introduced two new helpers that are used quite often throughout Agent 5:
`editFor` and `selectFor`. An `editFor` will create a label and input field for
the given property. Other options can be overriden and are documented in the
helper's file. The `selectFor` will create a label and dropdown menu for the
given property. It takes an additional piece of information, `listName`. This is
the name of the Clarify list the dropdown is based off of. With a little bit of
wiring up, Agent 5 will query the backend for that list and automatically
populate the dropdown with the correct values. As with the `editFor` helper, the
`selectFor` has other options that can be overridden and are documented in its
file.

### View

Now that we have a template, let's create a View that will be responsible for
rendering and handling interactions with it.

First, the Require.js setup:

```js
define([
hbs!newModemTpl // Special Require.js helper syntax
],
function(newModemTpl) {
  'use strict';
```

Next our view:

```js
  var NewModemView = Marionette.ItemView.extend({
    template: newModemTpl,
```

We need to associate an event (user clicking "Save" button) to an action:

```js
    events: {
      'click .btn-save-modem': 'save'
    },

    save: function(e) {
      e.preventDefault(); // Block browser from doing default action

      // Submit the form using the jquery.form plugin's 'ajaxSubmit' function
      this.$('#new-modem-form').ajaxSubmit({
        success: function() {
          // Do something when we get response back from server
        }
      });
    }
  });
```

Then create our standard controller and router and return the controller so that
require.js has an object to inject into other modules:

```js
  // Create basic controller that will render the view in 'show'
  var Controller = Marionette.Controller.extend({
    show: function(region) {
      var newModem = new NewModem();
      var newModemView = new NewModemView({ model: newModem });

      region.show(newModemView);

      return newModem;
    }
  });

  // Create a router to handle the new route
  var Router = Marionette.AppRouter.extend({
    appRoutes: {
      'modem/new': 'show'
    }
  });

  // Instantiate the controller and router
  var controller = new Controller();
  var router = new Router({ controller: controller });

  return controller;
});
```
