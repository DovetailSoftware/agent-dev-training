# Potpourri 

## Automation 101

* raking 
* releasing
* optimization

## Logging

* Logging is your friend. 
* Logging is easy.
* Logging saves your bacon.
* Logging config file

## Making a Toolkit

* Additional fields (only work on API's base table)
* Create a custom toolkit with DB transactionality

## Validation of Input Models

* Only works with POSTs
* Currently only server side
* link to fubuworld docs
* Apply [rules](http://fubuworld.com/fubuvalidation/rules/) to an input model validator
* Add input model in-place validation
* Create a custom rule
* Validation sources

## Adding nuget dependencies

```
ripple install Dovetail.SDK.ModelMap -p custom
```

## Bottles 101

Bottles are .Net assemblies that can register endpoints and static content to be served with your web application. 

We have found a lot of nice reuse by creating bottles for site features:

* Search - Conventional entity search
* Lists - Render dynamic regular and hierarchical lists on your pages
* Attachments - Support for uploading and downloading attachments 

We are recommending that you put all new functionality being added via Bottle to the Agent web application. 

### Assets (Things in your content folder)

One thing to know about bottles is that whenever you add an asset that will be packaged with the web application. An image, javascript, CSS, or template file you'll need to run the bottle command to ensure they are packaged up to be served by the web application.

This happens when you `rake` automatically. If you only want to do the bottle command on its own. You can run `rake bottle_{bottle_name}`. So for the custom bottle that would be 

```
rake custom_bottle
```

#### Content route

Anything found in the **content/{path}** directory of a bottle will be served up via the **_content/{path}** route off the root of the website.


#### Resource path redirecting to landing page

### Lifecycle

Bottles are found and initialized during application startup. Any any Bottle assembly found in your web application's `bin` folder will have their content "exploded" into the `Web/fubu-content`. **Important:** Your web application will need to have write access to this folder for this to work.

During development it is handy to see updates to assets you are changing without having to re-package your bottles everytime. This is accomplised by linking your bottle to your Web application using the a `.links` file found in your web application folder. 

```xml
<?xml version="1.0"?>
<links xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
  <include>..\Agent.Support</include>
  <include>..\Agent.Console</include>
  <include>..\Agent.Shared.Core</include>
  <include>..\custom</include>
</links>
```

If you add a bottle to your project you simply add a `<include/>` element to your `.links` file.
