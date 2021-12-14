## How to fetch data from the 3rd party api, save it in the database and make it avaiable within umbraco?

### Create an API Controller

- Create new class library project
- Remove the default Class1.cs
- Install correct version of UmbracoCms.Web, based on `website.core/packages.config`
  ![packages.config](https://i.imgur.com/O2svxDS.png)

- Create `Controllers/Api` directory
- Create a new class `YourControllerNameController` which extends UmbracoApiController

```C#
public class YourControllerNameController : UmbracoApiController
```

- In your controller create a Test endpoint
  ```C#
        [HttpGet]
        public async Task<IHttpActionResult> Test()
        {
            return Ok(DateTime.Now);
        }
  ```
- Ensure it can be hit on the standard routing
  default routing is `umbraco/api/ControllerNameWithoutController/ActionName`

- In the `Website/Dependencies/Projects` add references to the new project

---

### Create an API client

- Add a directory called `Clients` and create an Api Client class, which queries the external API to get data

- Add a directory `Models` and add models based on the data recieved from the api
  You can use `https://json2csharp.com/` to convert Json to C# Classes online
  E.G.:

```C#
  {
        public string Copyright { get; set; }
        public string Date { get; set; }
        public string Explanation { get; set; }
        public string Hdurl { get; set; }
        public string MediaType { get; set; }
        public string ServiceVersion { get; set; }
        public string Title { get; set; }
        public string Url { get; set; }
    }
```

- Add a method to your Api Client to retrieve the data using RestSharp
- Test it by returning the data from the API Endpoint

## Create custom table to store received data

- Create directory `CustomData` to contain all required info for additional data

- Create directories: `Migrations/Schemas`
- Create a class with inital schema based on the previously created model
  This will be used for the creation of our custom table. This is just the schema, Data Transfer Objects will be created at later stage.
- Inside `Migrations`, create a class `AddOurCustomDatabaseTableMigration` which extends MigrationBase and implemens the migration
  E.G:

```C#

public class AddOurCustomDatabaseTableMigration : MigrationBase
    {
        public AddOurCustomDatabaseTableMigration(IMigrationContext context) : base(context)
        {
        }

        public override void Migrate()
        {
            Logger.Debug(typeof(OurSchema), "Running migration {MigrationStep}", $"AddOurCustomDatabaseTableMigration");

            if (!TableExists("OurCustomDatabaseTableName"))
            {
                Create.Table<OurSchema>().Do();
            }
            else
            {
                Logger.Debug(typeof(OurSchema), "The database table {DbTable} already exists, skipping", "OurCustomDatabaseTableName");
            }
        }
    }

```

- Inside `CustomData` create directory `Components`
- Add a class which `IComponent` and injects the required services
- Execute the migration in this Component

```C#
public class OurCustomComponent : IComponent
    {
        private IScopeProvider _scopeProvider;
        private IMigrationBuilder _migrationBuilder;
        private IKeyValueService _keyValueService;
        private ILogger _logger;

        public BlogCommentsComponent(IScopeProvider scopeProvider, IMigrationBuilder migrationBuilder, IKeyValueService keyValueService, ILogger logger)
        {
            _scopeProvider = scopeProvider;
            _migrationBuilder = migrationBuilder;
            _keyValueService = keyValueService;
            _logger = logger;
        }

        public void Initialize()
        {
            // Create a migration plan for a specific project/feature
            // We can then track that latest migration state/step for this project/feature
            var migrationPlan = new MigrationPlan("NameForOurMigration");

            // This is the steps we need to take
            // Each step in the migration adds a unique value
            migrationPlan.From(string.Empty)
                .To<"AddOurCustomDatabaseTableMigration">("WriteShortDescribtionOfMigration");

            // Go and upgrade our site (Will check if it needs to do the work or not)
            // Based on the current/latest step
            var upgrader = new Upgrader(migrationPlan);
            upgrader.Execute(_scopeProvider, _migrationBuilder, _keyValueService, _logger);
        }


        public void Terminate()
        {
            throw new System.NotImplementedException();
        }
    }


```

- Inside `CustomData` create directory `Composers`
- Add a class called `OurCustomDatabaseTableComposer` which simply triggers the component by extending `ComponentComposer<T>``

```C#
public class OurCustomDatabaseTableComposer : ComponentComposer<OurCustomComponent>
    {
            // ComponentComposer<T>
            // It's an implementation of IComposer, that provides a quicker way to add a custom component to the Component's collection.
            // Creating a C# class that inherits from ComponentComposer<YourComponentType>
            // will automatically add YourComponentType to the collection of Components.
    }
```

- Run the web app, to check that the table is created!

---

### Create the Entity to use as the Data Transfer Object

- Create a directory `Entities`
- Add a class for `OurCustomDatabaseTableEntity`
- Copy the columns from our initial migration based on `DatabaseTableSchema`
- If you run other migrations don't forget to update entity with new fields

### Create a Service to update and add a new endpoint to Controller

- Create a directory `Services` and add new class for our `CustomDatabasaTableService`
- Inject required Umbraco classed, which are auto-registered with Dependecy Injection
- Write an `Upsert` method that takes one of the API models, and return an entity model
- Register the service with DI so it can be injected into the Controller. To do this, add a ServiceComposer to a new directory called `Composers` and register the service as well as your ApiController, so that thte Service can be injected
- Add an new endpoint to your Controller which saves the received data and returns the entity
