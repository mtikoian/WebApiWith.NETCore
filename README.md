# WebApi with .NET Core

Samples and resources of how to design WebApi with .NET Core

- [WebApi with .NET Core](#webapi-with-net-core)
  * [Project Configuration](#project-configuration)
  * [Routing](#routing)
    + [Route templates](#route-templates)
      - [Route parameters](#route-parameters)
      - [Route constraints](#route-constraints)
    + [Routing pipeline](#routing-pipeline)
    + [Routing with endpoints](#routing-with-endpoints)
    + [Routing with controllers](#routing-with-controllers)
      - [Conventional controllers routing](#conventional-controllers-routing)
      - [Routing with attributes](#routing-with-attributes)
    + [Links](#links)
  * [REST](#rest)
  * [API Versioning](#api-versioning)
  * [Filters](#filters)
  * [Middleware](#middleware)
  * [Inversion of Control and Dependency Injection](#inversion-of-control-and-dependency-injection)
  * [API Testing](#api-testing)
  * [Projects structure](#projects-structure)
  * [Logging](#logging)
    + [General](#general)
      - [Log Levels](#log-levels)
      - [Log Categories](#log-categories)
      - [Log Scopes](#log-scopes)
      - [Log Events](#log-events)
      - [Links](#links-1)
    + [Serilog](#serilog)
      - [Links](#links-2)
    + [NLog](#nlog)
      - [Links](#links-3)
    + [Elastic Stack - Kibana, LogStash etc.](#elastic-stack---kibana--logstash-etc)
      - [Links](#links-4)
  * [CorrelationId](#correlationid)
      - [Links](#links-5)
  * [Docker](#docker)
  * [CI/CD](#ci-cd)
  * [Storage](#storage)
    + [EntityFramework](#entityframework)
    + [Dapper](#dapper)
  * [Caching](#caching)
  * [GraphQL](#graphql)
  * [CQRS](#cqrs)
  * [OAuth](#oauth)

## Project Configuration

## Routing

From [the documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.1): *"Routing is responsible for matching incoming HTTP requests and dispatching those requests to the app's executable endpoints."*

Saying differently routing is responsible for finding exact endpoint based on the request parameters - usually based on the URL pattern matching.

Endpoint executes the logic that creates an HTTP response based on request.

To use routing and endpoints it's needed to call `UseRouting` and `UseEndpoints` extension method on app builder in `Startup.Configure` method. That will register routing in middleware pipeline.

Note that those methods should be registered in the order as presented above. If the order is changed then it won't be registered properly.

### Route templates

Templates add flexibility to supported URL definition.

The simplest option is **static URL** where you have just URL, eg:
- `/Reservations/List`
- `/GetUsers`
- `/Orders/ByStatuses/Closed`

#### Route parameters

Static URLs are fine for the list endpoints, but if we'd like to get a list of records.  
To allow dynamic matching (eg. reservation by Id) we need to use **parameters**. They can be added using `{parameterName}` syntax. eg.
- `/Reservations/{id}`
- `/users/{id}/orders/{orderId}`

They don't need to be only used instead of concrete URL part. You can also do eg.:
- `/Reservations?status={reservationStatus}&user={userId}` - this will get parameters from the query string and match eg. `/Reservations?status=Open&userId=123` and will have `status` parameter equal to `Open` and `userId` equal to `123`,
- `/Download/{fileName}.{extension}` - this will match eg. `/Download/testFile.txt` and end up with two route data parameters - `fileName` with `testFile` value and `extension` with `txt` accordingly,
- `/Configuration/{entityType}Dictionary` - this will match `/Configuration/OrderStatusDictionary` and will have `entityType` parameter with `OrderStatus` value.

You can also add **catch-all** parameters - `{**parameterName}`, that can be used as fallback when no route was found:
- `/Reservations/{id}/{**reservationPath}` - this will match eg. `/Reservations/123/changeStatus/confirmed` and will have `reservationPath` parameter with `changeStatus/confirmed` value

It's also possible to make the parameter optional by adding `?` after its name:
- `/Reservations/{id?}` - this will match both `/Reservations` and `/Reservation/123` routes

#### Route constraints

Route template parameters can contain **constraints** to narrow down the matched results. To use it you need to add constraint name after parameter name `{prameter:constraintName}`.
There is a number of predefined route constraints, eg:
- `/Reservations/{id:guid}` - will match eg. `/Reservations/632863d2-5cbf-4c9f-92e1-749d264d965e` but wont' match eg. `/Reservations/123`,
- `/Reservations/top/{limit:int:minlength(1):maxLength(10)` - this will allow to pass integers between `1` and `10` for `limit` parameter. So it will allow to get at most top 10 reservations,
- `/Inbox?from={fromEmailAddress:regex(\\[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,4})}` - regex can be also used to eg. check email address or provide more advanced format check. This will match `/Inbox?from=john.doe@company.com` and will have `fromEmailAddress` parameter with `john.doe@company.com` value,
- see more constraints examples in [route constraint documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.1#route-constraint-reference).

Note - failing constraint will result with `400 - BadRequest` status code, however, the messages are generic and not user friendly. So if you'd like to make them more related to your business case - it's suggested to do move it to validation inside the code.

You can also define your **custom constraint**. The sample use case would be when you want to provide the validation for your business id format.

See sample that validates if reservation id is built from 3 non-empty parts split by `|`;

```c#
public class ReservationIdConstraint : IRouteConstraint
{
    public bool Match(
        HttpContext httpContext,
        IRouter route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (routeKey == null)
        {
            throw new ArgumentNullException(nameof(routeKey));
        }

        if (values == null)
        {
            throw new ArgumentNullException(nameof(values));
        }

        if (!values.TryGetValue(routeKey, out var value) && value != null)
        {
            return false;
        }
        
        var reservationId = Convert.ToString(value, CultureInfo.InvariantCulture);
        
        return reservationId.Split("|").Where(part => !string.IsNullOrWhiteSpace(part)).Count() == 3;
    }
}
```

You need to register it in `Startup.ConfigureServices` in `AddRouting` method:

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // registers controllers in dependency injection container
        services.AddControllers();

        services.AddRouting(options =>
        {
            options.ConstraintMap.Add("reservationId", typeof(ReservationIdConstraint));
        });
    }

    // (...)
}
```

Then you can use it to in route:
- `/Reservations/{id:reservationId}` - this will match `/Reservations/RES|123|01` (and get `id` parameter with value `RES|123|01`) but wont't match `/Reservations/123`.

### Routing pipeline

Routing is split into the following steps:
- request URL parsing
- perform matching against registered routes (it's done in parallel, so the order of registration doesn't matter)
- from matching routes, remove all that do not match  routes constraints (eg. route parameter defined as int was not numeric)
- select single best matching (the most concrete one) if possible, from the left routes. If there are still more than one matches - the exception is being thrown. If there was only single match but value does not match constraint then exception will be thrown.

Having eg. following routes:

- `/Clients/List`
- `/Clients/{id}`
- `/Reservations/{id:alpha}`
- `/Reservations/{id:int}`
- `/Reservations/List`

and trying to match `/Reservation/List` the routing process will find matching templates so:

- `/Reservations/{id:alpha}`
- `/Reservations/{id:int}`
- `/Reservations/List`

It matched the `Reservations` part and then both `{id}` routes (as `List` could be just string id text) and concrete part `List`.

Then constraints will be verified and we'll end up with two routes (as `{id:int}` does not match because `List` is not an integer).

- `/Reservations/{id:alpha}`
- `/Reservations/List`

From this set both are matching, but `List` is more concrete.

Accordingly:
- trying to match `Reservations/abcde` routing will match `/Reservations/{id:alpha}` route,
- trying to match `Reservations/123` routing will match `/Reservations/{id:int}` route.
 
### Routing with endpoints

ASP.NET Core allows to define raw endpoints without the need to use controllers. They can be defined inside `UseEndpoints` method, by calling `UseGet`, `UsePost` etc. methods:

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // registers routing in middleware pipeline
        app.UseRouting();
        
        // defines endpoints to be routed
        app.UseEndpoints(endpoints =>
        {
             endpoints.MapGet("/Reservations/{id}", async context =>
             {
                 var name = context.Request.RouteValues["id"];
                 await context.Response.WriteAsync($"Reservation with {id}!");
             });
        });
    }
}
``` 

Using endpoints currently requires a lot of bare-bone code. This will change with .NET 5 where it will get a set of useful methods that will make it first-class citizen. See more in accepted API review: [link](https://github.com/dotnet/aspnetcore/issues/17160).

### Routing with controllers

Http requests can be mapped to controller with two ways: conventional and through attributes

#### Conventional controllers routing

Conventional is done by calling `MapControllerRoute` method inside `UseEndpoints`. It allows to provide route template (`pattern`), name and controller action mapping.

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // registers routing in middleware pipeline
        app.UseRouting();
        
        // defines endpoints to be routed
        app.UseEndpoints(endpoints =>
        {
            // defines concrete routing to single controller action
            endpoints.MapControllerRoute(name: "blog",
                pattern: "Reservations/{id}",
                defaults: new { controller = "Reservations", action = "Get" });
            
            // defines "catch-all" routing that will route all requests
            // matching `/Controller/Action` or `/Controller/Action/id`
            endpoints.MapControllerRoute(name: "default",
                pattern: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
``` 

Important thing to note is controllers should have the `Controller` suffix in the name (eg. `ReservationsController`), but routes should be defined without it (so `Reservations`).

#### Routing with attributes

Controllers are derived from the MVC pattern concept. They are responsible for orchestration between requests (inputs) and models. 
Routing can be defined by putting attributes on top of method and controller definition.

If you want to use Controllers then you should also call `AddControlers` in configure services (to register them in Dependency Container) and `MapControllers` inside `UseEndpoints` to map controllers routes configuration.

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // registers controllers in dependency injection container
        services.AddControllers();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // registers routing in middleware pipeline
        app.UseRouting();
        
        // defines endpoints to be routed
        app.UseEndpoints(endpoints =>
        {
            // maps controllers routes to endpoints
            endpoints.MapControllers();
        });
    }
}
```

**Route attribute**

The most generic attribute is `[Route]`. It routes that will direct to the method that it's marking.

```c#
public class ReservationsController : Controller
{
    [Route("")]
    [Route("Reservations")]
    [Route("Reservations/List")]
    [Route("Reservations/List/{status?}")]
    public IActionResult List(string status)
    {
        //(...)
    }

    [Route("Reservations/Summary")]
    [Route("Reservations/Summary/{userId?}")]
    public IActionResult Summary(int? userId)
    {
        // (...)
    }
}
```

In this example routes:
- `/`, `/Reservations`, `/Reservations/List`, `/Reservations/List/Open` will be routed to `List` method,
- `/Reservations/Summary`, `Reservations/Summary/123` will be routed to `Summary` method.

Important note is that you should not use `action`, `area`, `controller`, `handler`, `page` as route template variable (eg. `/Reservations/{page}`). Those names are reserved for the internals of routing logic. Using them will make routing fail.

**HTTP methods attributes**

ASP.NET Core provides also more specific attributes `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, `[HttpHead]`, `[HttpPatch]` representing HTTP methods. Besides the URL routing they also perform matching based on the HTTP method.
Normally using them you should add `[Route]` attribute on a controller that will add prefix for all the routes defined by HTTP verbs attributes.

Sample of the most common CRUD controller definition:

```c#
[Route("api/[controller]")]
[ApiController]
public class ReservationsController : ControllerBase
{
    [HttpGet]
    public IActionResult List([FromQuery] string filter)
    {
        //(...)
    }

    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        // (...)
    }
    
    [HttpPost]
    public IActionResult Create([FromBody] CreateReservation request)
    {
        // (...)
    }

    [HttpPut("{id}")]
    public IActionResult Put(int id, [FromBody] UpdateReservation request)
    {
        // (...)
    }

    [HttpDelete("{id}")]
    public IActionResult Delete(int id)
    {
        // (...)
    }
}
```

Using `[Route("api/[controller]")]` will define route based on the controller name - in this case it will be `/api/Reservations`. By convention WebApi routes usually start with a `/api` prefix. Prefix existence is optional and can have a different value. If you'd like you could also add suffix eg. `[Route("api/[controller]/open")]` if eg. you'd like to have dedicated controller for open reservations.
The benefit of using `[controller]` is that when you rename controller the route will be also updated. If you want to avoid accidental route name change then you should use concrete route eg. `[Route("api/reservations")]`

Having that:
- `GET /api/Reservations` will be routed to `List` method. Value for the `filter` parameter, because of `[FromQuery]` attribute will be mapped from request query string. For `GET /api/Reservations?filter=open` it will have `open` value, for default route `GET /api/Reservations` it will be `null`,
- `GET /api/Reservations/123` will be routed to `Get` method. Value of the `id` parameter will be taken by convention from the route parameter,
- `POST /api/Reservations/123` will be routed to `Create` method. Value for the `request` parameter, because of `[FromBody]` attribute will be mapped from request body (so eg. JSON sent from client),
- `PUT /api/Reservations/123` will be routed to `Update` method,
- `DELETE /api/Reservations/123` will be routed to `Delete` method. 

It's not mandatory to use route prefix. Most of the time it's useful, but when you have nesting inside the API then it's worth setting up it manually eg. 

```c#
[ApiController]
public class UserReservationsController : ControllerBase
{
    [HttpGet("api/users/{userId}/reservations")]
    public IActionResult List(int userId, [FromQuery] string filter)
    {
        //(...)
    }

    [HttpGet("api/users/{userId}/reservations/{id}")]
    public IActionResult Get(int userId, int id)
    {
        // (...)
    }
    
    [HttpPost("api/users/{userId}/reservations/{id}")]
    public IActionResult Create(int userId, [FromBody] CreateReservation request)
    {
        // (...)
    }

    [HttpPut("api/users/{userId}/reservations/{id}/status")]
    public IActionResult Put(int userId, int id, [FromBody] UpdateReservationStatus request)
    {
        // (...)
    }
}
```

### Links

- [Microsoft Documentation - Routing in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.1)
- [Microsoft Documentation - Routing to controller actions in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/routing?view=aspnetcore-3.1)
- [DotNetMentors - http://dotnetmentors.com/mvc/explain-asp-net-mvc-routing-with-example.aspx](http://dotnetmentors.com/mvc/explain-asp-net-mvc-routing-with-example.aspx)
- [StrathWeb - Dynamic controller routing in ASP.NET Core 3.0](https://www.strathweb.com/2019/08/dynamic-controller-routing-in-asp-net-core-3-0/)
- [Andrew Lock - Accessing route values in endpoint middleware in ASP.NET Core 3.0](https://andrewlock.net/accessing-route-values-in-endpoint-middleware-in-aspnetcore-3/)

## REST

## API Versioning

## Filters

## Middleware

## Inversion of Control and Dependency Injection

## API Testing

## Projects structure

## Logging

### General

#### Log Levels

By default in .NET Core there are six levels of logging (available through [LogLevel](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.loglevel?view=dotnet-plat-ext-3.1) enum):
- `Trace` (value `0`) - the most detailed and verbose information about the application flow, 
- `Debug` (`1`) - useful information during the development process (eg. local environment bug investigation),
- `Information` (`2`) - usually important information about the application flow that can be useful for diagnostics and flow, 
- `Warning` (`3`) - potential unexpected application event or error that's not blocking flow (eg. operation was successfully saved to the database but notification failed) or transient error occurred but was succeeded after retry), 
- `Error` (`4`) - unexpected application error - eg. no record found to update, database timeout, argument exception etc.,
- `Critical` (`5`) - informing about critical events that require immediate action like application or system crash, end of disk space or database in the irrecoverable state,
- `None` (`6`) - means no logs at all, used usually in the configuration to disable logging for selected category.

It's important to keep in mind that `Trace` and `Debug` should not be used on production, and should be used only for development/debugging purposes (`Trace` is by default disabled). 
Because of their characteristic, they may contain sensitive application information to be effective (eg. system secrets, PII/GDPR Data). Because of that, we need to be sure that on production environment they are disabled as that may end up with security leak. 
As they're also verbose, then keeping them on the production system may increase significantly cost of logs storage. Plus too many logs make them unreadable and hard to read.

#### Log Categories

Each logger instance needs to have an assigned category. Categories allow to group logs messages (as a category will be added to each log entry).
By convention category should be passed as the type parameter of [ILogger<T>](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger-1?view=dotnet-plat-ext-3.1). Usually it's the class that we're injecting logger, eg.

```c#
[Route("api/Reservations")]
public class ReservationsController: Controller
{
    private readonly ILogger<ReservationsController> logger;

    public ReservationsController(ILogger<ReservationsController> logger)
    {
        this.logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateReservationRequest request)
    {
        var reservationId = Guid.NewGuid();

        // (...) 

        logger.LogInformation("Created reservation with {ReservationId}", reservationId);
        
        
        return Created("api/Reservations", reservationId);
    }

}
```

Log category created with type parameter will contain full type name (so eg. `LoggingSamples.Controllers.ReservationController`).


It's also possible (however not recommended) to define that through [ILoggerFactory](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.iloggerfactory?view=dotnet-plat-ext-3.1) `CreateLogger(string categoryName)` method:

```c#
[Route("api/Reservations")]
public class ReservationsController: Controller
{
    private readonly ILogger logger;

    public ReservationsController(ILoggerFactory loggerFactory)
    {
        this.logger = logger.CreateLogger("LoggingSamples.Controllers.ReservationController");
    }
}
```

Categories are useful for searching through logs and diagnose issues. 
As mentioned in the previous section - it's also possible to define in different log levels for configuration.

Eg. if you have a default log level `Information` and you need to investigate issues occurring in a specific controller (eg. `ReservationsController`) then you can change the log level to `Debug` for a dedicated category.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "LoggingSamples.Controllers.ReservationController": "Debug"
    }
  }
}
```

Then for all categories but `LoggingSamples.Controllers.ReservationController` you'll have logs logged for Information and above (`Information`, `Warning`, `Error`, `Critical`) and for `LoggingSamples.Controllers.ReservationController` also `Debug`.

The other example is to disable logs from selected category - eg. 
- because you noticed that is logging some sensitive information and you need quickly to change that, 
- you want to mute some unimportant system logs,
- you want to make sure that logs from a specific category (eg. `LoggingSamples.Controllers.AuthenticationController`) won't be ever logged on prod.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "LoggingSamples.Controllers.AuthenticationController": "None"
    }
  }
}
```

#### Log Scopes

Besides categories, it's possible to define logging scopes. They allow having add set of custom information to each log entry.

Scopes are disabled by default - if you'd like to use them then you need to toggle them on in configuration:

```json
{
  "Logging": {
    "IncludeScopes": true,
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

Having that you can use [ILogger.BeginScope method](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger.beginscope?view=dotnet-plat-ext-3.1) to define one or more logging scopes.

The first potential use case is to always add entity type and identifier to all logs in business logic to not need to add it in each entry. Eg. reservation id during its update. You can also create nested scopes.

```c#
[HttpPut]
public async Task<IActionResult> Create(Guid id, [FromBody] UpdateReservationRequest request)
{
    using(logger.BeginScope("For {EntityType}", "Reservation")
    {
         using(logger.BeginScope("With {EntityId}", id)
         {    
              logger.LogInformation("Starting reservation update process for {request}", request);
              // (...)
         }
    }
    
    return OK();
}
```

You can create also scopes with aspect programming way - so eg. in middleware to inject scopes globally. 

An example would be injecting as logging scope information from request eg. client IP, user id.
 
Sample below shows how to inject [CorellationID](#correlationid) into logger scope.

```c#
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate next;
    private readonly ILogger logger;

    public CorrelationIdMiddleware(RequestDelegate next, ILoggerFactory loggerFactory)
    {
        this.next = next;
        logger = loggerFactory.CreateLogger<CorrelationIdMiddleware>();
    }

    public async Task Invoke(HttpContext context /* other scoped dependencies */)
    {
        var correlationID = Guid.NewGuid();

        using (logger.BeginScope($"CorrelationID: {CorrelationID}", correlationID))
        {
            await next(context);
        }
    }
}
```

#### Log Events

The other option for grouping logs is log events. They are used normally to group them eg. by purpose - eg. updating an entity, starting controller action, not finding entity etc.
To define them you need to provide a standardized list of int event ids. Eg.

```c#
public class LogEvents
{
    public const int InvalidRequest = 911;
    public const int ConflictState = 112;
    public const int EntityNotFound = 1000;
}
```

Sample usage:

```c#
[HttpPut]
public IActionResult Update([FromBody] UpdateReservation request)
{
    logger.LogInformation("Initiating reservation creation for {seatId}", request?.SeatId);

    if (request?.SeatId == null || request?.SeatId == Guid.Empty)
    {
        logger.LogWarning(LogEvents.InvalidRequest, "Invalid {SeatId}", request?.SeatId);

        return BadRequest("Invalid SeatId");
    }

    if (request?.ReservationId == null || request?.ReservationId == Guid.Empty)
    {
        logger.LogWarning(LogEvents.InvalidRequest, "Invalid {ReservationId}", request?.ReservationId);

        return BadRequest("Invalid ReservationId");
    }

    // (...)
    return Created("api/Reservations", reservation.Id);
}
```

#### Links
- [Microsoft Docs - Logging in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-3.1)
- [Microsoft Docs - High-performance logging with LoggerMessage in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/loggermessage?view=aspnetcore-3.1)
- [Steve Gordon - High-Performance Logging in .NET Core](https://www.stevejgordon.co.uk/high-performance-logging-in-net-core)
- [Software Engineering StackExchange - Benefits of Structured Logging vs basic logging](https://softwareengineering.stackexchange.com/questions/312197/benefits-of-structured-logging-vs-basic-logging)
- [Message Templates](https://messagetemplates.org/)
- [Andre Newman - Tools and Techniques for Logging Microservices ](https://dzone.com/articles/tools-and-techniques-for-logging-microservices-1)
- [Siva Prasad Rao Janapati - Distributed Logging Architecture for Microservices](https://dzone.com/articles/distributed-logging-architecture-for-microservices)
- [Szymon Warda - Stop trying to mock the ILogger methods](https://indexoutofrange.com/Stop-trying-to-mock-the-ILogger-methods/)
- [Andrew Lock - How to include scopes when logging exceptions in ASP.NET Core](https://andrewlock.net/how-to-include-scopes-when-logging-exceptions-in-asp-net-core/)
- [Rico Suter - Logging with ILogger in .NET: Recommendations and best practices](https://blog.rsuter.com/logging-with-ilogger-recommendations-and-best-practices/)
- [Stephen Cleary - Microsoft.Extensions.Logging](https://blog.stephencleary.com/2018/06/microsoft-extensions-logging-part-2-types.html)
- [Stephen Cleary - A New Pattern for Exception Logging](https://blog.stephencleary.com/2020/06/a-new-pattern-for-exception-logging.html)

### Serilog

#### Links
- [Serilog Documentation](https://serilog.net/)
- [Nicholas Blumhardt - Setting up Serilog in ASP.NET Core 3](https://nblumhardt.com/2019/10/serilog-in-aspnetcore-3/)
- [Ben Foster - Serilog Best Practices](https://benfoster.io/blog/serilog-best-practices/)
- [Alfus Jaganathan - Scoped logging using Microsoft Logger with Serilog in .Net Core Application](https://www.initpals.com/net-core/scoped-logging-using-microsoft-logger-with-serilog-in-net-core-application/)

### NLog

#### Links
- [NLog Documentation](https://nlog-project.org/)
- [NLog Wiki - How to use structured logging](https://github.com/NLog/NLog/wiki/How-to-use-structured-logging)

### Elastic Stack - Kibana, LogStash etc.

#### Links
- [HumanKode - Logging with ElasticSearch, Kibana, ASP.NET Core and Docker](https://www.humankode.com/asp-net-core/logging-with-elasticsearch-kibana-asp-net-core-and-docker)
- [Than Le - Building logging system in Microservice Architecture with ELK stack and Serilog .NET Core](https://medium.com/@letienthanh0212/building-logging-system-in-microservice-architecture-with-elk-stack-and-serilog-net-core-part-1-8fe2dfcf9e6f)
- [Marco de Sanctis - Monitor ASP.NET Core in ELK through Docker and Azure Event Hubs](https://medium.com/@marcodesanctis2/monitor-asp-net-core-in-elk-through-docker-and-azure-event-hubs-6e519249af61)
- [Microsoft Docs - Logging with Elastic Stack](https://docs.microsoft.com/en-us/dotnet/architecture/cloud-native/logging-with-elastic-stack)
- [Ali Mselmi - Structured logging with Serilog and Seq and ElasticSearch under Docker ](https://dev.to/hasdrubal/structure-logging-with-serilog-and-seq-and-elasticsearch-under-docker-16dk)
- [Logz.io - Complete Guide to ELK Stack](https://logz.io/learn/complete-guide-elk-stack/#elasticsearch)
- [Logz.io - Best practices for managing ElasticSearch indices](https://logz.io/blog/managing-elasticsearch-indices/)
- [Andrew Lock - Writing logs to Elasticsearch with Fluentd using Serilog in ASP.NET Core](https://andrewlock.net/writing-logs-to-elasticsearch-with-fluentd-using-serilog-in-asp-net-core/)
- [Elastic Documentation - Install ElasticSearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)
- [AWS User Group Bengaluru - Log analytics with ELK stack](https://www.slideshare.net/AWSUsersGroupBengalu/log-analytics-with-elk-stack)

## CorrelationId

#### Links
- [Steve Gordon - ASP.NET Core Correlation IDs](https://www.stevejgordon.co.uk/asp-net-core-correlation-ids)
- [Steve Gordon - CorrelationId NuGet Package](https://github.com/stevejgordon/CorrelationId)
- [Vicenç García - Capturing and forwarding correlation IDs in ASP.NET Core](https://vgaltes.com/post/forwarding-correlation-ids-in-aspnetcore/)
- [Vicenç García - Capturing and forwarding correlation IDs in ASP.NET Core, the easy way](https://vgaltes.com/post/forwarding-correlation-ids-in-aspnetcore-version-2/)

## Docker

## CI/CD

## Storage

### EntityFramework

### Dapper

## Caching

## GraphQL

## CQRS

## OAuth 


