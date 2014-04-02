#  Data Projection - Showing A Modem

Previously we introduced you to our backend stack and showed you how to added an endpoint action which creates a modem. In this tutorial we are going to dive deep into pulling data out of your Clarify/Dovetail system and exposing more endpoints for your front-end. We will:

- Provide an endpoint for viewing existing modems.
- Show how to project data using **ModelMaps** from the database into your view models
- Demonstrate a diagnostics and AJAX testing workflow.

## Show endpoint action

Let's add the following to method to the `ModemEndpoint` class.

```cs 

// GET /custom/modems/{Id}
public Modem get_custom_modems_Id(ShowModemRequest request)
{
	return new Modem();
}
```
Then we'll add this new input model.
```cs
public class ShowModemRequest : ICustomRequest
{
	public int Id { get; set; }
}
``` 

As before we've added a method whose name will describe the URL used to access it. The input model is simple with only an integer identifier for the modem. This Id property will make to the **table_modem.objid** database field. 

### Demonstration

Let's get this endpoint added to the application and verify it is setup correctly using FubuDiagnostics. [http://localhost:25120/_fubu/](http://localhost:25120/_fubu/)

We'll also exercise the endpoint using a Chrome extension called [Advanced REST Client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo?hl=en-US).

## Data Projection with Model Map

It is very common need, when using a MVC architecture, to get data from your database into view models. You'll then use these view models with a view engine or serialize them out as JSON to the front-end. We have a handy mechanism found in [Dovetail Bootstrap](http://dovetailsoftware.com/clarify/kmiller/2009/04/27/introducing-dovetail-datamap/) for doing this called **ModelMap**. 

### Model

First let's look at the Modem type we created last time.

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

### Map

A ModelMap is used to define a mapping from a database table(s) and fields to the properties of your view model. Take a look at this very simple example. 

```cs
public class ModemMap : ModelMap<Modem>
{
	protected override void MapDefinition()
	{
		FromTable("modem")
			.Assign(d => d.Id).FromIdentifyingField("objid")
			.Assign(d => d.DeviceName).FromField("device_name")
			.Assign(d => d.HostName).FromField("hostname")
			.Assign(d => d.Status).FromField("active");
			.Assign(d => d.DeviceType).FromField("device_type")
	}
}
```

Model map provides a [DSL](http://en.wikipedia.org/wiki/Domain-specific_language) to describe to Dovetail SDK how how want to populate your view model. In this case starting at table_modem. We `Assign` each property on the view model `FromField` on the modem table. 

Notice how the *Id* field's map is a bit different. `FromIdentifyingField` declares that this field is unique. In a bit we'll leverage this detail to lookup a modem by id. 

This model map is about as simple as they come. You can get a bit more complicated. 

```cs
//use multiple fields to populate one property (space delimitted)
.Assign(d=>d.Name).FromFields("first_name", "last_name") 

//Map a tinyint to a boolean
.Assign(d=>d.IsActive).BasedOnField("active").Do(f=>Convert.ToInt32(f) == 1) 
```

If you want to learn more you can always explore the DSL's options using Intellisense and take a look at other implementations of types which derive from `ModemMap<T>` checkout `ContactMap` for a good example.

### Details... Details...

You can only have one ModelMap per view model. If you want to have a different ModelMap for a given view model you can always create a derived type.

```cs
public class ModemAlternative : Modem { }
public class ModemAlternativeMap : ModelMap<ModemAlternative> { ... }
``` 

Your ModelMaps need to be registered with the IoC container. By default any maps found in the custom bottle are added. Here is how our IoC container [StructureMap](https://github.com/structuremap/structuremap) does that.

```cs
public class customStructureMapRegistry : Registry
{
	public customStructureMapRegistry()
	{
		Scan(scan =>
		{
			scan.TheCallingAssembly();
			scan.ConnectImplementationsToTypesClosing(typeof(ModelMap<>));
			scan.WithDefaultConventions();
		});
	}
}
```

Let's take a look at how you'd use this model map in your endpoint action.

## Show action using IModelBuilder<T>

To build a view model from one of your ModelMaps. You have your endpoint take a dependency on an `IModelBuilder<T>` with T being your view model.

```cs
public class ModemEndpoint
{
	private readonly IModelBuilder<Modem> _builder;
	private readonly IModemRepository _repository;

	public ModemEndpoint(IModelBuilder<Modem> builder, IModemRepository repository)
	{
		_builder = builder;
		_repository = repository;
	}

	public Modem get_custom_modems_Id(ShowModemRequest request)
	{
		return _builder.GetOne(request.Id);
	}

	//... other endpoint action(s)
}
```

This action will ask the builder to retrieve a Modem by the unique id field which we specified in the map.

### Beware of 404s

Note if the requested ID does not match any modems in the database the model builder will return null and in FubuMVC if you return null for an actions result the default behavior is to return a 404 HTTP Status (Not Found.)

## Demonstration

Create some modems using SQL

```sql
INSERT INTO table_modem VALUES(1, 'Smart Modem 1200', 'Hayes Compatible', 'active', 'akebono.stanford.edu', 1)
INSERT INTO table_modem VALUES(2, 'US Robotics 2400', 'USR', 'active', 'wilson.domain.com', 1)
INSERT INTO table_modem VALUES(3, 'Zoom 300', 'Hayes Compatible', 'active', 'xyzzy.com', 1)
```

Simulate an AJAX request using [Advanced REST Client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo?hl=en-US).

[http://localhost:25120/custom/modems/1](http://localhost:25120/custom/modems/1)

## Building Many Modems

You are not limited to returning one object from your projections. You can easily return all the modems in the system using `GetAll`

```cs
var allModems = _builder.GetAll();
var tenModems = _builder.GetAll(10);
```

An array of modems is returned. You almost never want get all of something or even a limited number of them with out a filter. 

### Get Modems With A Filter

You are not limited to filtering your model map projects by only unique Ids. You can filter by any fields on the root table of your map. 

The `Get` method of the builder accepts a [filter expression](https://support.dovetailsoftware.com/documentation/Dovetail%20SDK/3.3.1/html/#fcSDK~FChoice.Foundation.Filters.FilterExpression_members.html) which creates a Filter used by the underlying Dovetail SDK Generic. To construct a filter you'll need to know the database field name, an operator and a value to filter by. 

```cs
var hayesModems = _builder.Get(f=>f.StartsWith("device_name", 'hayes'))
```

In this exmple we are getting all modems whose 'device_name' field has a string that starts with `hayes`. 

There are many different [filter operators](https://support.dovetailsoftware.com/documentation/Dovetail%20SDK/3.3.1/html/#fcSDK~FChoice.Foundation.Filters.FilterExpression_members.html) you can use to compose the collection of models you are looking for. 

### Sorting 

Let's add a sorting directive to the map so that our modems come back sorted by device name.

```cs
public class ModemMap : ModelMap<Modem>
{
	protected override void MapDefinition()
	{
		FromTable("modem")
			.Assign(d => d.Id).FromIdentifyingField("objid")
			.Assign(d => d.DeviceName).FromField("device_name")
			.Assign(d => d.HostName).FromField("hostname")
			.Assign(d => d.Status).FromField("active")
			.Assign(d => d.DeviceType).FromField("device_type")
			.SortAscendingBy("devince_name");
	}
}
```

### Advanced Projection Features

Other advanced topics we have not talked about yet are: 

#### Traversing relations (ViaRelation)

Pull in fields from a realted object.

```
.ViaRelation("case_commit2case", kase => kase.Assign(d => d.CaseId).FromField("id_number"))
```
This example a committment is having it's CaseId populated by the related case's `id_number` field.

#### Adhoc relations (ViaAdhocRelation)

Useful when your root is `FromView(viewName)` to create "ad-hoc" relations between the view and a schema table.

```
.ViaAdhocRelation("user_id", "user", "objid", user => user
	.Assign(d => d.EmailSignature).FromField("x_signature")
)
```

In this example the view's `user_id` field is being related to the `table_user.objid`.

#### MapMany and MapOne	

You can have hierarchial view models. Here we populate a Contact's site roles.

```cs
.MapMany<ContactSite>().To(s => s.Sites).ViaRelation("contact2contact_role", role => role
	.Assign(d => d.IsPrimary).BasedOnField("primary_site").Do(s => s == "1")
	.Assign(d => d.RoleName).FromField("role_name")
	.Assign(d => d.RoleDatabaseIdentifier).FromField("objid")
	.Assign(d => d.PrimarySite).FromField("primary_site")
	.ViaRelation("contact_role2site", site => site
		.Assign(d => d.Name).FromField("name")
		.Assign(a => a.DatabaseIdentifier).FromField("objid")
	)
)
```

#### Embedded filters (FilteredBy)

You can specify default filters for any traversed map level.

```cs
public class UserOpenCaseListingMap : ModelMap<UserOpenCaseListing>
{
    private readonly ICurrentSDKUser _user;

    public UserOpenCaseListingMap(ICurrentSDKUser user)
    {
        _user = user;
    }

    protected override void MapDefinition()
    {
        FromTable("qry_case_view")
            .Assign(a => a.Id).FromField("id_number")
            .Assign(a => a.Title).FromField("title")
            .Assign(a => a.SiteName).FromField("site_name")
            .Assign(a=>a.ContactName).FromFields("first_name", "last_name")
            .Assign(a=>a.Severity).FromField("severity")
            .FilteredBy(f => f.And(f.Equals("owner", _user.Username), 
                                    f.NotStartsWith("condition","Closed")));
    }
}
```

The `FilteredBy` feature is rarely used by I wanted to include it so that you could also see dependency injection works for a `ModelMap`


### Details... Details...

What is going on under the hood here is that a ClarifyGeneric is created for the map's root table and also for all relations being traversed. When a filter is applied by the builder it is applied to the root table. Because of this **you can only filter by fields found on the root table's schema.** If you just have to filter on a field on a different table you'll likely want to use a view `FromView` as your map's root which contains all the fields you need.

## Homework

Create an endpoint that returns Modems for hosts at a given TLD domain.

Exercise this new endpoint using your favorite tool and send a screenshot of the response.
