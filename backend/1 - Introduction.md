Agent 5 Backend - Introduction
===============================

## Tools

Here is a brief summary from lowest level up for some of the tooling we have in place for Agent 5.

### Dovetail SDK 

Getting data into and out of a Clarify database is made easy using the tooling Dovetail provides. [Dovetail SDK](https://support.dovetailsoftware.com/selfservice/products/show/Dovetail%20SDK) helps greatly with doing data access and it's APIs provide equivalents to many of  Clarify common operations. 

### Dovetail Bootstrap

[Dovetail Bootstrap](https://github.com/DovetailSoftware/dovetail-bootstrap) sits on top of the SDK and provides common tooling for building web applications and pulling data out of the Clarify database into Models for your favorite MVC framework. 

### FubuMVC

[FubuMVC](http://fubuworld.com) (Fubu) is a model view controller architecture that leans heavily conventions to shape how you development web applicaitons. Luckily it is also extremely customizable and we've used it with great success to make a flexible testable web applications. Fubu bets big on using [IoC containers](http://martinfowler.com/articles/injection.html) and we feel it pays off by making our code more modular and testable.

### Bottles

Bottles are a packaging mechanism for content and behavior used in Fubu web applications. You can package up features or entire parts of your website into Bottles assemblies. This capability allows you to partiion your application nicely and potentially benefit from reuse as Bottles can be used in more than one web application. 

## Single Page Architecture (SPA)

Our web application uses what is call a [Single Page Architecture](http://en.wikipedia.org/wiki/Single-page_application) or SPA. In reality our applicaiton is currently 2 SPAs:
* Console - Work finder and preview, search
* Support - case, subcase, solution, site, contact, employee

### Project Organization

We've split up our applicaiton into bottles aligned with our SPAs

 - Shared.Core - Shared content bottle
 - Console - SPA bottle 
 - Support - SPA bottle
 - Web - FubuMVC Web project

The Web project simply takes a dependency on the console and support bottles. Without them there would not be much going on other than authentication.

### Landing Pages

We have a rich frontend architecture that is not usually responsible for sending HTML to the user but we still need to serve up the shell of a landing page for each SPA. 

You will typically not need to worry about these pages in your customization efforts but it is good to know they exist. 

We use the [Spark View Engine](http://sparkviewengine.com) for doing our server side HTML presentation. Look for `.spark` files in the solution to see which content is rendered server side.

## Endpoints 

An endpoint is a server side action (C# method) that gets bound to a route, addressed by URL, used to get data into and out of your web application. 

### Endpoint Convention 

Endpoints are easy to create using a simple convention. 

```cs 
public class ModelEndpoint 
{
  // creates a route : GET /spa/modem/{Id}
  public ShowModemModel get_spa_modem_Id(ShowModemRequest request) { ... }
}
public class ShowModemRequest 
{ 
  public string Id { get; set; }
}

public class ShowModemModel { ... }
```
Here are the basics of the convention.
* Create a class whose name ends with Endpoint.
* Add a public method whose name will encode the http verb and route.
  * Separate the route components with underscores "_"
  * The first word defines the HTTP verb (GET, POST, PUT)
  * Each subsequent word is a route segment
  * Capitalized route segments will be matched with a property on the input model and turned into a route input.

### One Model In - One Model Out

FubuMVC is quite flexible but it has one very strong opinion. There is a primary convention called **One Model In and One Model Out**. Your actions have a single argument which is your input model and return type which is your output model. You'll never see an action with multiple arguments. 

Using this convention fubu executes your actions when an incoming web request matches your action's input model. Here is the basic execution of a fubu web request.

* Fubu will create an input model based on the web request and give it to your action. 
* Your action will transform the input model into an output model. 
* What Fubu does with that output model depends on how your applicaiton is configured. For us typically the output model will be returned as JSON to the browser. 

#### Input model 

One core tennet is that each action you expose to the world via an endpoint **needs to have a unique input model**. Why? Input models are often used to identify an route/action to be executed and many more interesting advanced things we won't talk about for now.

Data gets marshalled from the components of each web request into your action's input model. FubuMVC is responsible for this has has a powerful data binding mechanism that takes data from query string, post data, route data, even HTTP headers to populate the properties of your Action's input model. 

In the example above the GET action for showing modems had a very simple input model whose Id property was set by a route input. For POST actions that create objects you will have much more complicated input models. 

##### Basic Validation Example 
You can even annotate your input models and get automatic basic validation:

```cs
public class CreateModemRequest
{
    [Required, SchemaField("modem", "device_name")]
  public string DeviceName { get; set; }

  [SchemaField("modem", "active")]
  public string Status { get; set; }

  [BasedOnList("MODEM_TYPE")]
  public string DeviceType { get; set; }

  [BasedOnList("HOSTNAME")]
  public string HostName { get; set; }
}
```

An action which takes this class as an input model in Agent 5 will get ensure that the properties marked **Required** are set and that the ones with **SchemaField** do not have lengths too long for the schema to handle. 

#### Output model

Typically in our world the action's output model will be serialized as JSON and sent to the browser. 

Another more traditional possibility is that there is a (spark) view whose model matches the output model of your action the view will be rendered using your action's output model.

Fubu will use content negotiation to determine what format the output model will be returned in. This format depends on the `Accept` HTTP header. For this reason if you create an endpoint to test it using your browser you'll need to use something like [Postman](https://chrome.google.com/webstore/detail/postman-rest-client/fdmmgilgnpjigdojojpjoooidkmcomcm?hl=en) or I prefer [Advanced REST Client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo?utm_source=ha-en-na-us-webapp-Advanced%20Rest%20Client%20Application).  

##Custom SPA

We've created a place for your customizations to go where they can live in a seperate SPA than the included Dovetail Agent application. A large advantage of this is that it separates your code from the baseline code. This space is for introducing new entities and user interfaces to the application. If you wish to modify existing views you may still modify the bottles shipped with Agent 5.

### Tour

There are two projects under the source directory. **custom** and **custom.Testing** which are a bottle and a test project respectively. 

```txt
C:\projects\agent\source>dir custom*

Directory of C:\projects\agent\source

03/28/2014  03:04 PM    <DIR>          custom
03/28/2014  03:04 PM    <DIR>          custom.Testing
```

The custom bottle project has been setup to reference things you'd commonly need for creating new web experiences using Dovetail SDK and our Bootstrap, Agent 5 infrastructure.

The test project is a place for you to put your unit and integration [NUnit](http://www.nunit.org/) tests. 

### Bookmarkable Routes  

It is common in web applicaitons to have routes (URLs) to parts of your web application which people want to bookmark or share links. SPA based applications are no different but because most routes are "played back" on the client side, they usually have no server side equivalent. We have a solution for this. 

Marking any action's InputModel with a specific [marker interface](http://en.wikipedia.org/wiki/Marker_interface_pattern) will make that endpoint automatically transfer GET requests from user agents that `Accept: text/html` to your SPA's landing page.

#### Example

An AJAX request to the "Show Modem" route **GET /custom/modems/{Id}** returns a JSON blob of data for the application to render about the given modem. You'd also want this route to be bookmarkable and show the case view when a web browser visits it. To make sure this happens the input model of the show case endpoint is marked with `ICustomRequest` which enables a behavior in FubuMVC which will transfer non-AJAX GET requests to a specified landing endpoint.

Here is the FubuMVC policy configuration for the Custom Bottle which enables this behavior.

```cs
Policies.Add<TransferNonAjaxGETRequestsTo<ICustomRequest, LandingRequest>>();
```

Basically this says transfer requests marked with `ICustomRequest` to the endpoint whose input model is `LandingRequest`. This is another example of identifying target actions by their input model.

## Demonstration 

Let's work through a real world use case. To mesh well with our Front End discussion we'll create an endpoint which will receive a POST request for creating and viewing a Modem.

### Creating a Modem

We've already talked about the endpoint convention but let's review. To add an action to the applicaiton we just need to create a class ending in `Endpoint` and put public methods on this class whose name will define routes (urls) exposed to the world.

#### A Modem Endpoint
```cs
public class ModemEndpoint
{
  public AjaxContinuation post_custom_modems_create(CreateModemRequest request)
  {
      return new AjaxContinuation();
  }
}

public class CreateModemRequest 
{
    [Required]
  public string DeviceName { get; set; }
  public string DeviceType { get; set; }
  public string HostName { get; set; }
}
```

Having the class above in your project will add a POST endpoint `/custom/modems/create`. The CreateModemRequest type defines the values expected by the request. Validation for the required DeviceName property will be enforced.

#### Domain

Let's create a proper **Modem** [domain](http://en.wikipedia.org/wiki/Domain_model) object for us to transform this request into and save into our database. 

```cs
public class Modem 
{
    public int Id { get; set; }
  public string DeviceName { get; set; }
  public string HostName { get; set; }
  public string Status { get; set; }
  public string DeviceType { get; set; }
}
```

#### Repository 

Modem is a pretty simple object. Next, let's create a repository which will be responsible for creating and updating modem objects. 

```cs
public interface IModemRepository
{
    Modem Save(Modem modem);
}

public class ModemRepository : IModemRepository
{
  private readonly IClarifySession _session;

  public ModemRepository(IClarifySession session)
  {
    _session = session;
  }

  public Modem Save(Modem modem)
  {
    var dataset = _session.CreateDataSet();
    var modemGeneric = dataset.CreateGeneric("modem");

    ClarifyDataRow modemRow;
    if (modem.Id <= 0)
    {
      modemRow = modemGeneric.AddNew();
    }
    else
    {
      modemGeneric.Filter(f => f.Equals("objid", modem.Id));
      modemGeneric.Query();
      if(modemGeneric.Count < 1) 
                throw new ArgumentException("Modem ID {0} does not exist.".ToFormat(modem.Id));
      modemRow = modemGeneric[0];
    }

    modemRow["active"] = modem.Status;
    modemRow["device_name"] = modem.DeviceName;
    modemRow["device_type"] = modem.DeviceType;
    modemRow["hostname"] = modem.HostName;

    modemRow.Update();

    modem.Id = modemRow.DatabaseIdentifier();

    return modem;
  }
}
```
The repository is where we group all the **Modem** data access code we are going to write. Notice that the same **Save** method can handle updating existing modem objects.

#### Bring it all together

Now we will take a dependency on this repository and use it in our endpoint's action to **Save** the modem. 

```cs
public class ModemEndpoint
{
  private readonly IModemRepository _repository;

  public ModemEndpoint(IModemRepository repository)
  {
    _repository = repository;
  }

  public AjaxContinuation post_custom_modems_create(CreateModemRequest request)
  {
    var modem = new Modem
    {
      DeviceName = request.DeviceName,
      HostName = request.HostName,
      DeviceType = request.DeviceType,
      Status = "active"
    };
    var result = _repository.Save(modem);

    return new AjaxContinuation().ForSuccess(result).AddMessage("Modem created.");
  }
}
```

Now we have some working code creating a `Modem` and saving it which you can invoke from a web request. 

#### Ajax Continuations

The AjaxContinuation we've returned in the previous example an interesting feature we use extensively. For the example above the front end gets back a message like this: 

```js
{ 
    success: true, 
    target: { deviceName: 'my device', hostName: 'google.com', status: 'active', deviceType: '2400 baud'},
    message: 'Modem created.'
}
```

The front end consumes this object and can show a notificaiton and update the view's client side model. If not successful an error message is usually conveyed and can also be displayed as a notification.

This same mechanism is how validation errors are returned to the front end.


## Homework

We've covered a lot here from the basic tooling of Dovetail SDK to do data access to FubuMVC to create endpoint actions to deliver your application data to the front end. 

You might have noticed that the **IModemRepository** can also update existing Modem entities. **Your homework is to create an end point for editting a modem.** 

Remember your input model needs to be unique. What challenge does this present?
