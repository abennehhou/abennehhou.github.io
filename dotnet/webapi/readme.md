# Web Api

[Home](../../readme.md) > [.NET](../readme.md) > [.Web Api](./readme.md)

## Table of Contents

1. [Logging and data correlation](#logging-and-data-correlation)
    * [Initialize logging](#initialize-logging)
    * [Log calls](#log-calls)
    * [Data correlation](#data-correlation)

## Logging and data correlation

### Initialize logging

Here, we use `Log4Net` and `Common.Logging`.

First, use the following nuget packages

```xml
<package id="Common.Logging" version="3.4.1" targetFramework="net461" />
<package id="Common.Logging.Core" version="3.4.1" targetFramework="net461" />
<package id="Common.Logging.Log4Net208" version="3.4.1" targetFramework="net461" />
<package id="log4net" version="2.0.8" targetFramework="net461" />
```
In Web.config file, add the configSections, common and log4net sections:

```xml

<configSections>
<sectionGroup name="common">
    <section name="logging" type="Common.Logging.ConfigurationSectionHandler, Common.Logging" />
</sectionGroup>
<section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net" />
</configSections>
<common>
<logging>
    <factoryAdapter type="Common.Logging.Log4Net.Log4NetLoggerFactoryAdapter, Common.Logging.Log4net208">
    <arg key="configType" value="INLINE" />
    </factoryAdapter>
</logging>
</common>
<log4net>
  <appender name="RollingLogFileAppender" type="log4net.Appender.RollingFileAppender">
    <file value="D:\MyApi\Logs\MyApi.log" />
    <appendToFile value="true" />
    <rollingStyle value="Composite" />
    <datePattern value="yyyy-MM-dd" />
    <PreserveLogFileNameExtension value="true" />
    <maxSizeRollBackups value="90" />
    <lockingModel type="log4net.Appender.FileAppender+MinimalLock" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="date=%utcdate{yyyy-MM-ddTHH:mm:ss.fffZ};thread=%thread;level=%-5level;logger=%logger{1};%message%newline" />
    </layout>
  </appender>
  <root>
    <level value="ALL" />
    <appender-ref ref="RollingLogFileAppender" />
  </root>
</log4net>

```

After that, we will be able to use a logger. First, initialize a variable

```csharp
public static readonly ILog Logger = LogManager.GetLogger(typeof(MyClass));

public void MyMethod()
{
    Logger.Debug("My message.");
}
```

### Log calls

To log incoming calls, create a new message handler for logging (to be added in `WebApiConfig` file).


```csharp
config.MessageHandlers.Add(new LoggingMessageHandler());
```

This handler logs some information about the call, when it is not related to swagger documentation: it logs the request uri, request method, response code and elapsed time.

```csharp

public class LoggingMessageHandler : DelegatingHandler
{
    private static readonly ILog Logger = LogManager.GetLogger<LoggingMessageHandler>();
    private const string SwaggerUrl = "/swagger";

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var requestUri = request.RequestUri.ToString();

        // Ignore help pages
        if (!requestUri.ToLower().Contains(SwaggerUrl))
            return await base.SendAsync(request, cancellationToken);

        var stopwatch = Stopwatch.StartNew();
        var responseCode = string.Empty;

        try
        {
            var response = await base.SendAsync(request, cancellationToken);
            responseCode = response.StatusCode.ToString();

            return response;
        }
        finally
        {
            stopwatch.Stop();
            Logger.Debug($"RequestMethod={request.Method};RequestUri={requestUri};ResponseCode={responseCode};Elapsed={stopwatch.Elapsed}");
        }
    }
}

```

Example of the logged message: `RequestMethod=GET;RequestUri=http://localhost/myapi/api/foo/123;ResponseCode=OK;Elapsed=00:00:00.2123028`.

### Data correlation

Sometimes, we want to correlate data in the logs between many applications. For example: the identifier of a user, a session or an operation. This information can be provided in the header. We use thread context provided by log4net.

We can change the handler declared above to inject this data.

```csharp

public class LoggingMessageHandler : DelegatingHandler
{
    private static readonly ILog Logger = LogManager.GetLogger<LoggingMessageHandler>();

    private const string CorrelationIdHeaderName  = "x-correlation-id";
    private const string RequesterIdHeaderName    = "x-requester-id";
    private const string SessionIdHeaderName      = "x-session-id";

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        ...
        var correlationId = GetHeaderValue(request, CorrelationIdHeaderName) ?? request.GetCorrelationId().ToString();
        log4net.LogicalThreadContext.Properties["CorrelationId"] = correlationId;
        log4net.LogicalThreadContext.Properties["RequesterId"] = GetHeaderValue(request, RequesterIdHeaderName);
        log4net.LogicalThreadContext.Properties["SessionId"] = GetHeaderValue(request, SessionIdHeaderName);

        try
        {
            ...
            response.Headers.Add(CorrelationIdHeaderName, correlationId);
            ...
        }
        finally
        {
            ...
        }
    }

    private string GetHeaderValue(HttpRequestMessage request, string hearderKey)
    {
        return request.Headers.Where(x => x.Key == hearderKey).SelectMany(x => x.Value).FirstOrDefault();
    }
}

```
Then we change the conversion pattern to include the added properties:

```xml
<conversionPattern value="date=%utcdate{yyyy-MM-ddTHH:mm:ss.fffZ};thread=%thread;level=%-5level;logger=%logger{1};correlation=%property{CorrelationId};requester=%property{RequesterId};session=%property{SessionId};%message%newline" />
```

To log a whitespace instead of `(null)` when data is not provided, add this value in app settings.

```xml
<appSettings>
    <add key="log4net.NullText" value=" " />
</appSettings>
```

Example of the logged message: 

```
date=2018-05-07T10:49:41.597Z;thread=173;level=DEBUG;logger=MyClass;correlation=ae36a207-f6e6-4170-81a1-b86a2a17970d;requester=myUserLogin;session=my_session_62e73796-0694-4349-867f-b1aa8cc43d87;my message here
```

Example when information is not provided:

```
date=2018-05-05T22:47:53.208Z;thread=1;level=INFO ;logger=Startup;correlation= ;requester= ;session= ;Starting Api...
```
