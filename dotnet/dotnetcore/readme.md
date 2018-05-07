# ASP.NET Core

[Home](../../readme.md) > [.NET](../readme.md) > [.NET Core](./readme.md)

## Table of Contents

1. [Migration from Web Api 2 to .NET Core 2](#migration-from-web-api-2-to-dotnet-core-2)
    * [Migrate dependencies to .net standard](#migrate-dependencies-to-dotnet-standard)
    * [Create Api project](#create-api-project)
    * [Config files and logging](#config-files-and-logging)
    * [Dependency injection](#dependency-injection)
    * [Swagger and Api Versioning](#swagger-and-api-versioning)
    * [Fluent validation](#fluent-validation)
    * [Custom model binding](#custom-model-binding)
        * [Model Binder](#model-binder)
        * [Input Formatter](#input-formatter)
    * [CORS](#cors)
    * [Middlewares](#middlewares)
        * [Exception handling](#exception-handling)
        * [Logging](#logging)
    * [Changes in controller](#changes-in-controller)
2. [Inheritance](#inheritance)
    * [Return derived classes](#return-derived-classes)
    * [Add derived classes in input](#add-derived-classes-in-input)
    * [Add derived classes in documentation](#add-derived-classes-in-documentation)
    * [Validate derived classes](#validate-derived-classes)
3. [Exception Management](#exception-management)

## Migration from Web Api 2 to DotNet Core 2

Before migrating to .Net Core, you need to check that all nuget packages and libraries used in the project are availale in .Net standard.
Also, some features are not availale in .Net Core, for example message security in wcf, [see details here](https://github.com/dotnet/wcf/blob/master/release-notes/SupportedFeatures-v2.0.0.md).

Here are te steps I followed to migrate an Api from a Web Api 2 to .Net Core 2 project.

### Migrate dependencies to dotnet standard

First, all dependencies must be migrated from .Net Framework 4.6.X to .Net Standard 2.0.
Make sure that the sdk and runtime are installed. They are available [here](https://www.microsoft.com/net/download/visual-studio-sdks).

Two ways to migrate:
* Create a new project .net standard project, add nuget package, move the c# files from the old project, remove the old project.
* Change the csproj file: remove everything and replace it with this example:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <AssemblyName>[YourAssemblyName]</AssemblyName>
    <RootNamespace>[YourRootNamespace]</RootNamespace>
  </PropertyGroup>
</Project>
```

Then add nuget packages. Nuget packages will be added under `<PropertyGroup>` section, in `<ItemGroup>`.
Example:

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Logging" Version="2.0.0" />
    <PackageReference Include="MongoDB.Bson" Version="2.5.0" />
    <PackageReference Include="X.PagedList" Version="7.2.2" />
  </ItemGroup>
```

### Create Api project

Create a new project that will replace the current web api.
* Choose ASP.NET core web application > Web Api
* Add dependencies to .net standard libraries

### Config files and logging

Configuration is available in _appsettings.json_ file. Copy from old _Web.config_ file to _appsettings.json_ file.
In this example, I have one connection string, two app settings, and I use common.logging with log4net.
In .Net Core, I switched to the default logging _Microsoft.Extensions.Logging_ with _NLog_, using a _nlog.config_ file in the same level as _appsettings.json_ file.

Before:

```xml
<configSections>
    <sectionGroup name="common">
      <section name="logging" type="Common.Logging.ConfigurationSectionHandler, Common.Logging"/>
    </sectionGroup>
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
  </configSections>
<appSettings>
    <add key="Key1" value="Value1" />
    <add key="Key2" value="Value2" />
</appSettings>
<connectionStrings>
    <add name="Name1"
         connectionString="Connection1" />
</connectionStrings>
<log4net configSource="...."/>
```

After:

```json
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Trace",
      "System": "Warning",
      "Microsoft": "Warning"
    }
  },
  "Key1": "Value1",
  "Key2": "Value2",
  "Name1": "Connection1"
}
```

To access configuration, there is no more `ConfigurationManager`. You can access it from `Startup.cs` file.
Example: `var appSettingsValue = Configuration[AppSettingsKey];`.


To configure _NLog_ with _Microsoft.Extensions.Logging_, update the _Program.cs_ .

``` csharp
public static void Main(string[] args)
{
    // NLog: setup the logger first to catch all errors
    var logger = LogManager.LoadConfiguration("nlog.config").GetCurrentClassLogger();
    try
    {
        logger.Debug("init main");
        BuildWebHost(args).Run();
    }
    catch (Exception ex)
    {
        //NLog: catch setup errors
        logger.Error(ex, "Stopped program because of exception");
        throw;
    }
}

public static IWebHost BuildWebHost(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .UseStartup<Startup>()
        .ConfigureLogging((hostingContext, logging) =>
        {
            logging.ClearProviders();
            logging.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Trace);
        })
        .UseNLog()  // NLog: setup NLog for Dependency injection
        .Build();
}
```

Example of _nlog.config_ file:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      internalLogLevel="Warn"
      internalLogFile="D:\BSIC_DATA\MyApi\Logs\internal-nlog.log">
    <!-- the targets to write to -->
    <targets>
        <!-- write logs to file -->
        <target xsi:type="File" name="allfile" fileName="D:\BSIC_DATA\MyApi\Logs\All-${shortdate}.log"
                layout="${date:universalTime=True:format=yyyy-MM-ddTHH\:mm\:ss.fff}|${uppercase:${level}}|${logger}|${message} ${exception}" />
        <!-- another file log, only own logs. Uses some ASP.NET core renderers -->
        <target xsi:type="File" name="ownFile" fileName="D:\BSIC_DATA\MyApi\Logs\MyApi-${shortdate}.log"
                layout="${date:universalTime=True:format=yyyy-MM-ddTHH\:mm\:ss.fff}|${uppercase:${level}}|${logger}|${message} ${exception}" />
    </targets>
    <!-- rules to map from logger name to target -->
    <rules>
        <!-- Ignore trace Microsoft logs in allFile -->
        <logger name="Microsoft.*" maxlevel="Trace" final="true" />
        <logger name="*" minlevel="Trace" writeTo="allfile" />
        
        <!-- Ignore all Microsoft logs in ownFile -->
        <logger name="Microsoft.*" final="true" />
        <logger name="*" minlevel="Trace" writeTo="ownFile" />
    </rules>
</nlog>

```

Then, we can inject loggers, for example `ILogger<MyService>`.
More details [here](https://github.com/NLog/NLog.Web/wiki/Getting-started-with-ASP.NET-Core-2).

### Dependency injection

There is no need to use _Unity_ for dependency injection. We can use the provided one. Example: `services.AddTransient<IFooRepository, FooRepository>(c => new FooRepository(myconnectionString));`.


### Swagger and Api Versioning

Here is an example of what we could have in Web Api 2:

```csharp
// Add Versioning and versioned documentation using swagger
config.AddApiVersioning(
        o =>
        {
            o.AssumeDefaultVersionWhenUnspecified = true;
            o.DefaultApiVersion = new ApiVersion(1, 0);
            o.ReportApiVersions = true;
        }
    );
var apiExplorer = config.AddVersionedApiExplorer(o => o.GroupNameFormat = "F");
var virtualPath = HostingEnvironment.ApplicationHost.GetVirtualPath();
config.EnableSwagger(
    SwaggerRootTemplate,
    swagger =>
    {
        swagger.RootUrl(req => req.RequestUri.GetLeftPart(UriPartial.Authority) + req.GetConfiguration().VirtualPathRoot.TrimEnd('/') + virtualPath);
        swagger.IncludeXmlComments(XmlCommentsPath);
        swagger.MultipleApiVersions(
            (apiDescription, version) => apiDescription.GetGroupName() == version,
            info =>
            {
                foreach (var group in apiExplorer.ApiDescriptions)
                {
                    info.Version(group.Name, $"My API {group.ApiVersion}");
                }
            });
    })
    .EnableSwaggerUi(swagger =>
    {
        swagger.EnableDiscoveryUrlSelector();
        swagger.DisableValidator();
    });
```

In .Net core, the syntax is different and is done in two steps:
* In ConfigureServices method:

```csharp
// Add Versioning and versioned documentation using swagger
services.AddMvcCore().AddVersionedApiExplorer(o => o.GroupNameFormat = "F");
services.AddMvc();

services.AddApiVersioning(
    o =>
    {
        o.AssumeDefaultVersionWhenUnspecified = true;
        o.DefaultApiVersion = new ApiVersion(1, 0);
        o.ReportApiVersions = true;
    });

// Register the Swagger generator, defining one or more Swagger documents
services.AddSwaggerGen(c =>
{
    var provider = services.BuildServiceProvider().GetRequiredService<IApiVersionDescriptionProvider>();

    foreach (var description in provider.ApiVersionDescriptions)
    {
        c.SwaggerDoc(description.GroupName, new Info()
            {
                Title = $"My API {description.ApiVersion}",
                Version = description.ApiVersion.ToString()
            });
    }

    // Set the comments path for the Swagger JSON and UI.
    var basePath = AppContext.BaseDirectory;
    c.IncludeXmlComments(Path.Combine(basePath, XmlComments));
});
```

* In Configure method:

```csharp
// Enable middleware to serve generated Swagger as a JSON endpoint.
app.UseSwagger();

// Enable middleware to serve swagger-ui (HTML, JS, CSS, etc.), specifying the Swagger JSON endpoint.
app.UseSwaggerUI(c =>
{
    foreach (var description in provider.ApiVersionDescriptions)
    {
        c.SwaggerEndpoint($"{description.GroupName}/swagger.json", description.GroupName.ToUpperInvariant());
    }
});
```

### Fluent validation

Registering fluent validation is different between Web Api 2 and .Net core. In Web Api 2:

```csharp
var validators = AssemblyScanner.FindValidatorsInAssemblyContaining<MyValidator>();
validators.ForEach(validator => unityContainer.RegisterType(validator.InterfaceType, validator.ValidatorType, new HierarchicalLifetimeManager()));
FluentValidationModelValidatorProvider.Configure(config, provider =>
{
    provider.ValidatorFactory = new UnityValidatorFactory(unityContainer);
});
```

In .Net Core:

```csharp
services.AddMvc().AddFluentValidation(fv =>
{
    fv.RegisterValidatorsFromAssemblyContaining<MyValidator>();
    fv.ImplicitlyValidateChildProperties = true;
}
);
```

To return a bad request when the validation fails, create a _ValidateCommand_ attribute and add it to the operations you want to validate:


```csharp

    /// <summary>
    /// Validates model state before executing the method.
    /// </summary>
    public class ValidateCommandAttribute : ActionFilterAttribute
    {

        /// <summary>
        /// Occurs before the action method is invoked.
        /// </summary>
        /// <param name="actionContext"> The action context. </param>
        public override void OnActionExecuting(ActionExecutingContext actionContext)
        {
            if (!actionContext.ModelState.IsValid)
            {
                var controller = actionContext.Controller as ControllerBase;
                if (controller != null)
                {
                    actionContext.Result = controller.BadRequest(actionContext.ModelState);
                    return;
                }
            }
            base.OnActionExecuting(actionContext);
        }
    }

```

### Custom model binding

#### Model Binder

Example for a custom binding, used to bind comma separated values to a list of strings.

* In Web Api 2, we used _IModelBinder_.

```csharp
public class MyListBinder : IModelBinder
{
    public bool BindModel(HttpActionContext actionContext, ModelBindingContext bindingContext)
    {
        var value = bindingContext.ValueProvider.GetValue(bindingContext.ModelName);
        var result = new List<string>();
        if (!string.IsNullOrEmpty(value?.AttemptedValue))
        {
            var values = value.AttemptedValue.Split(new[] { "," }, StringSplitOptions.RemoveEmptyEntries);
            result.AddRange(values);
        }
        bindingContext.Model = result;

        return true;
    }
}
```

And it is declared in WebApiConfig.cs file: `config.BindParameter(typeof(IList<string>), new MyListBinder());`

* In .Net core, we need a model binder and a model binder provider.

```csharp
public class MyListBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        if (bindingContext == null)
            throw new ArgumentNullException(nameof(bindingContext));

        var modelName = bindingContext.ModelName;
        var valueProviderResult = bindingContext.ValueProvider.GetValue(modelName);

        if (valueProviderResult == ValueProviderResult.None || valueProviderResult.Length == 0)
            return Task.CompletedTask;

        var model = valueProviderResult.Values
            .SelectMany(x => x?.Split(new[] { "," }, StringSplitOptions.RemoveEmptyEntries))
            .Where(y => !string.IsNullOrEmpty(y))
            .Distinct()
            .ToList();

        bindingContext.Result = ModelBindingResult.Success(model);

        return Task.CompletedTask;
    }
}

public class MyListBinderProvider : IModelBinderProvider
{
    public IModelBinder GetBinder(ModelBinderProviderContext context)
    {
        if (context == null)
            throw new ArgumentNullException(nameof(context));

        if (context.Metadata.ModelType == typeof(IList<string>))
            return new MyListBinder();

        return null;
    }
}
```

And it is declared in Startup.cs file: `services.AddMvc(options => { options.ModelBinderProviders.Insert(0, new MyListBinderProvider());});`

#### Input formatter

Example for a custom parsing of request's body.

* In Web Api, we can use this custom parsing with a custom attribute [FromMyCustomBody] instead of [FromBody]:

```csharp
 public class MyCustomBodyModelBinder : HttpParameterBinding
    {
        public MyCustomBodyModelBinder(HttpParameterDescriptor descriptor) : base(descriptor)
        {
        }

        public override Task ExecuteBindingAsync(ModelMetadataProvider metadataProvider, HttpActionContext actionContext, CancellationToken cancellationToken)
        {
            var binding = actionContext.ActionDescriptor.ActionBinding;

            var content = actionContext.Request.Content;

            return content.ReadAsStringAsync().ContinueWith(task =>
            {
                var json = task.Result;
                var bindingParameter = binding.ParameterBindings.OfType<MyCustomBodyModelBinder>().FirstOrDefault();
                if (bindingParameter != null)
                {
                    var type = bindingParameter.Descriptor.ParameterType;
                    var name = bindingParameter.Descriptor.ParameterName;
                    var converted = CustomConvert(json, type);

                    SetValue(actionContext, converted);
                    var modelMetadataProvider = Descriptor.Configuration.Services.GetModelMetadataProvider();
                    var validator = Descriptor.Configuration.Services.GetBodyModelValidator();
                    validator.Validate(converted, type, modelMetadataProvider, actionContext, name);                    
                }
            });
        }

        public override bool WillReadBody => true;
    }
}

[AttributeUsage(AttributeTargets.Class | AttributeTargets.Parameter, Inherited = true, AllowMultiple = false)]
public sealed class FromMyCustomBodyAttribute : ParameterBindingAttribute
{
    public override HttpParameterBinding GetBinding(HttpParameterDescriptor parameter)
    {
        if (parameter == null)
            throw new ArgumentNullException(nameof(parameter));

        return new MyCustomBodyModelBinder(parameter);
    }
}
```

* In .Net core, we create an input formatter and declare it in the startup. `services.AddMvc(options => { options.InputFormatters.Insert(0, new MyCustomInputFormatter()); });`. Then we keep using [FromBody] attribute.

```csharp
public class MyCustomInputFormatter : InputFormatter
{
    public MyCustomInputFormatter()
    {
        SupportedMediaTypes.Add("application/json");
    }
    public override async Task<InputFormatterResult> ReadRequestBodyAsync(InputFormatterContext context)
    {
        var request = context.HttpContext.Request;
        using (var reader = new StreamReader(request.Body))
        {
            var content = await reader.ReadToEndAsync();
            var type = context.ModelType;
            var converted = CustomConvert(content, type);
            return await InputFormatterResult.SuccessAsync(converted);
        }
    }
    protected override bool CanReadType(Type type)
    {
        return type.Assembly == typeof(MyType).Assembly;
    }
}
```

### CORS

In Web Api, to enable CORS fo everyone: `config.EnableCors(new EnableCorsAttribute("*", "*", "*"));`.
In .Net Core, it is done in two parts: `services.AddCors();` and `app.UseCors(builder => builder.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod());`.

### Middlewares

#### Exception handling

Example in WebApi 2, an exception filter is declared: `config.Filters.Add(new GlobalExceptionFilter());`

```csharp
public class GlobalExceptionFilter : ExceptionFilterAttribute
{
    public override void OnException(HttpActionExecutedContext context)
    {
        var exception = context.Exception as ApiException;
        var httpError = new HttpError(...) {....};
        var statusCode = ....;
        context.Response = context.Request.CreateErrorResponse(statusCode, httpError);
    }
}
```

In .Net Core, a middleware is declared: `app.UseMiddleware(typeof(ErrorHandlingMiddleware));`

```csharp
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;

    public ErrorHandlingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
       var result = JsonConvert.SerializeObject(new { Error = exception.Message, ... });
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = ...;
        return context.Response.WriteAsync(result);
    }
}
```

#### Logging

* In Web Api 2, we create a delegating handle and declare it in the startup. `config.MessageHandlers.Add(new LoggingMessageHandler());`

```csharp
public class LoggingMessageHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var requestUri = request.RequestUri.ToString();
        var stopwatch = Stopwatch.StartNew();
        string responseContent = null;
        string statusCode = null;
        string statusReason = null;

        try
        {
            var response = await base.SendAsync(request, cancellationToken);

            responseContent = response.Content == null ? null : await response.Content.ReadAsStringAsync();

            statusCode = ((int)response.StatusCode).ToString();
            statusReason = response.ReasonPhrase;

            return response;
        }
        finally
        {
            stopwatch.Stop();
            var elapsed = stopwatch.Elapsed.ToString();
            var requestContent = request.Content == null ? null : await request.Content.ReadAsStringAsync();
            //Log here
        }
    }
}
```

* In .Net Core, we use a middleware `app.UseMiddleware(typeof(LoggingMessageMiddleware));`, to be declared before exception handling.

```csharp

public class LoggingMessageMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public LoggingMessageMiddleware(RequestDelegate next, ILogger<LoggingMessageMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        var request = context.Request;
        var requestUri = request.GetDisplayUrl();
        var stopwatch = Stopwatch.StartNew();
        string statusCode = null;

        try
        {
            await _next(context);
            var response = context.Response;
            statusCode = response.StatusCode.ToString();
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogDebug(....);
        }
    }
}
```

### Changes in controller

* `[RoutePrefix("api/foo")]` -> `[Route("api/foo")]`
* `[ResponseType(typeof(Foo))]` -> `[ProducesResponseType(typeof(PagedListDto<Foo>), 200)]`
* `IHttpActionResult` -> `IActionResult`
* `Request.RequestUri` -> `Request.GetDisplayUrl()`
* For swagger, http method must be declared for each action. Add missing `[HttpGet]`
* For swagger, ignore actions using the attribute ` [ApiExplorerSettings(IgnoreApi = true)]`


## Inheritance

### Return derived classes

Let's start with this basic example: we want to get a list of vehicles.

```csharp

        [HttpGet]
        [ProducesResponseType(typeof(List<Vehicle>), 200)]
        [Route("", Name = RouteNameSearch)]
        public async Task<IActionResult> GetVehiclesAsync()
```

This will return a list of objects having only the properties declared in _Vehicle_ class. To be able to get properties declared in sub-classes, the _Vehicle_ class needs to know its inherited classes:

```csharp

    [KnownType(typeof(Bike))]
    [KnownType(typeof(Car))]
    public class Vehicle

```

### Add derived classes in input

Example: we want to post a vehicle.

```csharp

        [HttpPost]
        [ValidateCommand]
        [ProducesResponseType(typeof(Vehicle), 201)]
        [Route("")]
        public async Task<IActionResult> Post([FromBody] Vehicle vehicle)

```

To be able to accept a derived class, we need to have a custom json formatter and declare it in _Startup.cs_ file.

* First, create a converter that converts a Vehicle to its derived class.

```csharp

    /// <summary>
    /// Converter used to parse a vehicle.
    /// </summary>
    public class VehicleConverter : JsonConverter
    {
        /// <summary>
        /// Determines whether this instance can convert the specified object type.
        /// </summary>
        public override bool CanConvert(Type objectType)
        {
            return typeof(Vehicle).GetTypeInfo().IsAssignableFrom(objectType);
        }

        /// <summary>
        /// Gets a value indicating whether this Newtonsoft.Json.JsonConverter can write.
        /// </summary>
        public override bool CanWrite => false;

        /// <summary>
        /// Reads the JSON representation of the object.
        /// </summary>
        public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
        {
            if (reader.TokenType == JsonToken.Null)
                return null;

            var item = JObject.Load(reader);
            var vehicleType = item["VehicleType"]?.ToString(); // Here, we assume that vehicle has a property called VehicleType containing vehicle type.
            switch (vehicleType)
            {
                case "Bike":
                    return item.ToObject<Bike>();
                case "Car":
                    return item.ToObject<Car>();
                default:
                    throw new ArgumentException($"Unknown vehicle type '{vehicleType}'");
            }
        }

        /// <summary>
        /// Writes the JSON representation of the object.
        /// </summary>
        public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
        {
            //Not used because CanWrite is set to false
        }
    }


```

* Then create a formatter that uses this converter.

```csharp

    public class BodyInheritanceInputFormatter : InputFormatter
    {
        public BodyInheritanceInputFormatter()
        {
            SupportedMediaTypes.Add("application/json");
        }
        public override async Task<InputFormatterResult> ReadRequestBodyAsync(InputFormatterContext context)
        {
            var request = context.HttpContext.Request;
            using (var reader = new StreamReader(request.Body))
            {
                var content = await reader.ReadToEndAsync();
                var type = context.ModelType;

                var converters = new JsonConverter[] { new VehicleConverter() }; // Add other converters here

                var converted = JsonConvert.DeserializeObject(content, type, converters);
                return await InputFormatterResult.SuccessAsync(converted);
            }
        }
        protected override bool CanReadType(Type type)
        {
            return type.Assembly == typeof(Vehicle).Assembly;
        }
    }

```

* And finally use this formatter in startup file

```csharp

            services.AddMvc(options =>
            {
                options.InputFormatters.Insert(0, new BodyInheritanceInputFormatter()); // Add custom formatter to parse body into derived class
            })
            .AddJsonOptions(options => options.SerializerSettings.NullValueHandling = NullValueHandling.Ignore) // Ignore null values in response

```

### Add derived classes in documentation

To include them in the swagger documentation, add this swagger gen option: `c.DocumentFilter<PolymorphismDocumentFilter>();` and define a new filter:


```csharp

public class PolymorphismDocumentFilter : IDocumentFilter
{
    private static void RegisterSubClasses(ISchemaRegistry schemaRegistry)
    {
        var assembly = typeof(Vehicle).Assembly;
        var allTypes = assembly.GetTypes();

        var allBaseClassDtoTypes = allTypes
                                    .Where(x => x.IsDefined(typeof(KnownTypeAttribute), false))
                                    .ToList();

        var derivedTypes = allTypes
            .Where(x => allBaseClassDtoTypes.Any(abstractType => abstractType != x && abstractType.IsAssignableFrom(x)))
            .ToList();

        foreach (var item in derivedTypes)
        {
            schemaRegistry.GetOrRegister(item);
        }
    }

    public void Apply(SwaggerDocument swaggerDoc, DocumentFilterContext context)
    {
        RegisterSubClasses(context.SchemaRegistry);
    }
}

```


### Validate derived classes

To validate derived classes with fluent validation, first, create a base class for validators, that includes a method that can be used to add validation rules for derived classes:


```csharp

public class ValidatorBase<TBase> : AbstractValidator<TBase>
{
    public void MapDerivedValidator<TType, TValidatorType>() where TValidatorType : IEnumerable<IValidationRule>, IValidator<TType>, new() where TType : TBase
    {
        When(t => t.GetType() == typeof(TType), () => AddDerivedRules<TValidatorType>());
    }

    private void AddDerivedRules<T>() where T : IEnumerable<IValidationRule>, new()
    {
        IEnumerable<IValidationRule> validator = new T();
        foreach (var rule in validator)
        {
            AddRule(rule);
        }
    }
}

```

Then, in the validator for the parent class, include validator for sub-classes.

```csharp

public class VehicleValidator : ValidatorBase<Vehicle>
{
    public VehicleValidator()
    {
        RuleFor(request => request.VehicleType)
            .NotEmpty();

        // Add rules for common properties here

        MapDerivedValidator<Bike, BikeValidator>();
    }
}

public class BikeValidator : ValidatorBase<Bike>
{
    public BikeValidator()
    {
        // Add rules for properties specific to a bike
    }
}

```

## Exception Management

To share api error codes with external applications (clients) instead of error messages, create an enumeration with all the error codes and define exceptions using these error codes. Here is a dummy example:

```csharp

public enum ApiErrorCode
{
    InternalError,
    UserNotFound,
    UserAlreadyExists,
    UserWrongPassword,
    UserDisabled
}

public class ApiException : Exception
{
    public ApiErrorCode ApiErrorCode { get; set; }

    public ApiException(ApiErrorCode errorCode, string message, Exception innerException = null)
        : base(message, innerException)
    {
        ApiErrorCode = errorCode;
    }
}

```

You can also create other exceptions inheriting from `ApiException` and that will be used to return an accurate http error code.
Here, for example, we create two classes: `ValidationApiException` and `ResourceNotFoundApiException`.
The `ValidationApiException` will be thrown when data in the input is invalid, and `ResourceNotFoundApiException` when the resource we are looking for is not found.

```csharp

public class ValidationApiException : ApiException
{
    public ValidationApiException(ApiErrorCode errorCode, string message, Exception innerException = null)
        : base(errorCode, message, innerException)
    {
    }
}
```

Example of use:

```csharp

[HttpGet]
[ProducesResponseType(typeof(UserDto), 200)]
[Route("{id}", Name = RouteNameGetById)]
public async Task<IActionResult> GetAsync(string id)
{
    if (string.IsNullOrEmpty(id))
        throw new ValidationApiException(ApiErrorCode.MissingInformation, $"Parameter {nameof(id)} must be provided.");

    var user = await _userService.GetUserByIdAsync(id);

    if (user == null)
        throw new ResourceNotFoundApiException(ApiErrorCode.UserNotFound, $"Cannot find user with id=\"{id}\"");

    var userDto = Mapper.Map<UserDto>(user);

    return Ok(userDto);
}

```


Then, define a middleware for exception handling. Here, the exception is logged, and the http status code is determined by the exception type. The ApiErrorCode is returned in the response.

```csharp

public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public ErrorHandlingMiddleware(RequestDelegate next, ILogger<ErrorHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        var code = HttpStatusCode.InternalServerError; // 500 if unexpected
        _logger.LogWarning(exception.Message);
        _logger.LogTrace($"Stacktrace: {exception.StackTrace}");
        while (exception.InnerException != null)
        {
            exception = exception.InnerException;
            _logger.LogWarning($"Inner exception: {exception.Message}");
            _logger.LogTrace($"Stacktrace: {exception.StackTrace}");
        }

        var apiException = exception as ApiException;
        if (apiException != null)
            code = GetHttpStatusCodeFromException(apiException);

        var apiErrorCode = apiException?.ApiErrorCode ?? ApiErrorCode.InternalError;

        var result = JsonConvert.SerializeObject(new { Error = exception.Message, ApiErrorCode = apiErrorCode.ToString() });
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = (int)code;
        return context.Response.WriteAsync(result);
    }

    private HttpStatusCode GetHttpStatusCodeFromException(ApiException exception)
    {
        if (exception is ResourceNotFoundApiException)
            return HttpStatusCode.NotFound;

        if (exception is ValidationApiException)
            return HttpStatusCode.BadRequest;

        // Add here other exceptions

        return HttpStatusCode.InternalServerError;
    }
}

```

Finally, use this middleware in `Startup.cs` file.

```csharp
app.UseMiddleware(typeof(ErrorHandlingMiddleware));
```

