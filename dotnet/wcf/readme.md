# WCF

[Home](../../readme.md) > [.NET](../readme.md) > [WCF](./readme.md)

## Table of Contents

1. [Behaviors](#behaviors)
    * [Transform fault exceptions](#transform-fault-exceptions)
    * [Client logging](#client-logging)
2. [Tips](#tips)
    * [Extend partial classes](#extend-partial-classes)
    * [Dependency injection](#dependency-injection)
    * [Closing connections](#closing-connections)


## Behaviors

### Transform fault exceptions

To transform fault exceptions that are provided by an external wcf service:

* Create a client message inspector.

```csharp

public class ExceptionHandlingInspector : IClientMessageInspector
{
    private const string FaultDetailsElementName = "Detail";

    private static readonly ILog Logger = LogManager.GetLogger<ExceptionHandlingInspector>();

    private readonly DataContractSerializer _serializer;

    public ExceptionHandlingInspector()
    {
        _serializer = new DataContractSerializer(typeof(MyFaultException));
    }

    public object BeforeSendRequest(ref Message request, IClientChannel channel)
    {
        return Guid.NewGuid();
    }

    public void AfterReceiveReply(ref Message reply, object correlationState)
    {
        if (reply.IsFault)
        {
            // Create a copy of the original reply to allow default WCF processing
            var buffer = reply.CreateBufferedCopy(Int32.MaxValue);

            // Create a copy to work with
            var newCopy = buffer.CreateMessage();

            // Restore the original message 
            reply = buffer.CreateMessage();

            var exception = ReadFaultDetail(newCopy) as Exception;

            if (exception != null)
            {
                throw exception;
            }
        }
    }

    private object ReadFaultDetail(Message reply)
    {
        using (XmlDictionaryReader reader = reply.GetReaderAtBodyContents())
        {
            // Find <soap:Detail>
            while (reader.Read())
            {
                if (reader.NodeType == XmlNodeType.Element && string.Equals(FaultDetailsElementName, reader.LocalName))
                {
                    break;
                }
            }
            if (reader.EOF)
            {
                return null;
            }

            // Move to the contents of <soap:Detail>
            if (!reader.Read())
            {
                return null;
            }

            // Deserialize the fault
            try
            {
                var myFaultException = _serializer.ReadObject(reader) as MyFaultException;

                if (myFaultException != null)
                {
                    return new MyException(myFaultException.SomeProperty);
                }
            }

            catch (SerializationException ex)
            {
                Logger.Warn(ex);

                return null;
            }

            return null;
        }
    }
}

```

* Create a behavior extension.

```csharp

public class ExceptionHandlingBehavior : BehaviorExtensionElement, IEndpointBehavior
{
    protected override object CreateBehavior()
    {
        return new ExceptionHandlingBehavior();
    }

    public override Type BehaviorType
    {
        get { return typeof(ExceptionHandlingBehavior); }
    }

    public void Validate(ServiceEndpoint endpoint)
    {
    }

    public void AddBindingParameters(ServiceEndpoint endpoint, BindingParameterCollection bindingParameters)
    {
    }

    public void ApplyDispatchBehavior(ServiceEndpoint endpoint, EndpointDispatcher endpointDispatcher)
    {
    }

    public void ApplyClientBehavior(ServiceEndpoint endpoint, ClientRuntime clientRuntime)
    {
        clientRuntime.MessageInspectors.Add(new ExceptionHandlingInspector());
    }
}

```

Add the behavior to service model

```xml

 <system.serviceModel>
    <behaviors>
        <endpointBehaviors>
            <behavior name="MyBehavior">
                <MyExceptionHandling />
            </behavior>
        </endpointBehaviors>
    </behaviors>
    <bindings>
    </bindings>
    <client>
        <endpoint ... behaviorConfiguration="MyBehavior" />
    </client>
    <extensions>
      <behaviorExtensions>
        <add name="MyExceptionHandling" type="MyNamespace.ExceptionHandlingBehavior, MyAssembly" />
      </behaviorExtensions>
    </extensions>
  </system.serviceModel>

```

### Client logging

We want to be able to log requests, responses and elapsed time. We can create a behavior for this and use it like: `<WcfMessageLoggerEndpointBehavior LogRequest="true" LogResponse="false" />`.

* Create the behavior element.

```csharp

public class WcfMessageLoggerEndpointBehavior : BehaviorExtensionElement, IEndpointBehavior
{
    public override Type BehaviorType
    {
        get { return typeof (WcfMessageLoggerEndpointBehavior); }
    }
    [ConfigurationProperty("LogRequest")]
    public bool LogRequest
    {
        get { return (bool)this["LogRequest"]; }
        set { this["LogRequest"] = value; }
    }

    [ConfigurationProperty("LogResponse")]
    public bool LogResponse
    {
        get { return (bool)this["LogResponse"]; }
        set { this["LogResponse"] = value; }
    }

    public WcfMessageLoggerEndpointBehavior()
    {
    }

    public WcfMessageLoggerEndpointBehavior(bool logRequest, bool logResponse)
    {
        LogRequest  = logRequest;
        LogResponse = logResponse;
    }

    public void Validate(ServiceEndpoint endpoint)
    {
    }

    public void AddBindingParameters(ServiceEndpoint endpoint, BindingParameterCollection bindingParameters)
    {
    }

    public void ApplyDispatchBehavior(ServiceEndpoint endpoint, EndpointDispatcher endpointDispatcher)
    {
    }

    public void ApplyClientBehavior(ServiceEndpoint endpoint, ClientRuntime clientRuntime)
    {
        clientRuntime.MessageInspectors.Add(new WcfMessageLoggerClientInspector(LogRequest, LogResponse));
    }

    protected override object CreateBehavior()
    {
        return new WcfMessageLoggerEndpointBehavior(LogRequest, LogResponse, ResponseTimeLimit);
    }

}

``` 

* Create the client inspector.

```csharp

public class WcfMessageLoggerClientInspector : IClientMessageInspector
{
    private static readonly ILog Logger = LogManager.GetLogger<WcfMessageLoggerClientInspector>();
    
    private static readonly int bufferSize = int.MaxValue;

    private readonly ConcurrentDictionary<Guid, DateTime> _startTimeByRequestCorrelationId;

    private string _requestAction;

    public bool LogRequest { get; private set; }

    public bool LogResponse { get; private set; }

    public WcfMessageLoggerClientInspector(bool logRequest, bool logResponse)
    {
        LogRequest  = logRequest;
        LogResponse = logResponse;        
    }

    public object BeforeSendRequest(ref Message request, IClientChannel channel)
    {
        var correlationId = Guid.NewGuid();

        _startTimeByRequestCorrelationId[correlationId] = DateTime.UtcNow;

        _requestAction = request.Headers.Action ?? "NONE";

        if (LogRequest)
        {
            var copy = request.CreateBufferedCopy(bufferSize);
            request = copy.CreateMessage();
            var message = GetMessageAsString(copy.CreateMessage());
            Logger.Debug($"WCF request: {message}");
        }

        return correlationId;
    }

    public void AfterReceiveReply(ref Message reply, object correlationState)
    {
        Guid correlationId;
        Guid.TryParse(correlationState.ToString(), out correlationId);

        if (LogResponse)
        {
            var copy = reply.CreateBufferedCopy(bufferSize);
            reply = copy.CreateMessage();
            var message = GetMessageAsString(copy.CreateMessage());
            Logger.Debug($"WCF reply: {message}");
        }

        DateTime startDateTime;
        if (_startTimeByRequestCorrelationId.TryRemove(correlationId, out startDateTime))
        {
            var elapsedTime = DateTime.UtcNow - startDateTime;
            Logger.Debug($"Elapsed time for '{_requestAction}': {elapsedTime}.");
        }
    }

    private static string GetMessageAsString(Message message)
    {
        var builder = new StringBuilder();
        using (var sWriter = new StringWriter(builder))
        {
            using (var xWriter = XmlWriter.Create(sWriter, new XmlWriterSettings { Indent = false, OmitXmlDeclaration = true }))
            {
                message.WriteMessage(xWriter);
            }
        }

        return builder.ToString();
    }
}

```

Remark: WCF calls can be logged in services like for clients. Use a service behavior and a dispatch message inspector.

## Tips

### Extend partial classes

We can extend partial classes for example to provide new constructors. This is useful when we want to add parameters to the constructor or fill properties with default values (example: generate a request id, a date).

Define the partial class with the same namespace as the one defined in the service reference.

### Dependency injection


* Create inteface: `IClientChannelFactory<out TChannel> ` with operations Open, Close, Abort and CreateChannel.
* Creates its implementation `ClientChannelFactory<TChannel> : IClientChannelFactory<TChannel> where TChannel : IClientChannel` with a constructor having in parameter `endpointConfigurationName` and with a private field `ChannelFactory<TChannel> factory`. Use this factory in all implementation.
* Create a Mock implementation with fake implementation

```csharp
public class ClientChannelFactoryMock<TChannel> : IClientChannelFactory<TChannel> where TChannel : IClientChannel, new()
{
    public ClientChannelFactoryMock(string endpointConfigurationName)
    {

    }
    public void Open()
    {
    }

    public void Close()
    {
    }

    public void Abort()
    {
    }

    public TChannel CreateChannel()
    {
        return new TChannel();
    }
}

```
* Inject the ClientChannelFactory in the service you use: `public MyServices(IClientChannelFactory<IMyChannel> myChannelFactory` and use it: create the channel, do operation, close it. If there is an exception, abort it.
* Create a mock of the channel: `MyChannelMock : IMyChannel` and create fake implementation of its operations.
* How to switch registration between real implementation and mock in unity dependency injection container

```csharp

if (isMockused)
{
    container.RegisterType<IClientChannelFactory<IMyChannel>, ClientChannelFactoryMock<MyChannelMock>>(
            new PerThreadLifetimeManager(),
            new InjectionFactory(c => new ClientChannelFactoryMock<MyChannelMock>(endpointName)));
}
else
{
    container.RegisterType<IClientChannelFactory<IMyChannel>, ClientChannelFactory<IMyChannel>>(
            new PerThreadLifetimeManager(),
            new InjectionFactory(c => new ClientChannelFactory<IMyChannel>(endpointName)));
}

```
* For unit tests, use `moq` to mock all the calls to myChannelFactory.


More info provided in [this](https://megakemp.com/2009/07/02/isolating-wcf-through-dependency-injection-part-ii/) website.

### Closing connections

Usually, the WCF services are called this way:

```csharp
using (var client = new MyClient())
{
    var myresult = client.DoAction(myParameter);
}
```

The using statement ensures that Dispose is called even if an exception occurs while you are calling methods on the object. 
In `ClientBase.cs` class, the Dispose method callse the Close method.

When the server throws an unhandled exception, the client is in a faulted state. In this case, the `client.Close()` throws the exception: `The communication object, System.ServiceModel.Channels.ServiceChannel, cannot be used for communication.`. We must call `client.Abort()` instead of `client.Close()`.

To avoid this problem, we can transform the code as below:

```csharp
var client = new MyClient();
try
{
    var myresult = client.DoAction(myParameter);
}
finally
{
    if (communicationObject == null || communicationObject.State == CommunicationState.Closed)
        return;
 
    if (communicationObject.State == CommunicationState.Faulted)
        communicationObject.Abort();
    else
        communicationObject.Close();
}
```

To make this code common to all WCF calls, create an extension method.

```csharp

public static class CommunicationObjectExtensions
{
    private static readonly Common.Logging.ILog Logger = Common.Logging.LogManager.GetLogger<ICommunicationObject>();

    public static void CallAndClose(this ICommunicationObject client, Action action)
    {
        try
        {
            action();
        }
        finally
        {
            CloseOrAbortServiceChannel(client);
        }
    }

    public static async Task CallAndCloseAsync(this ICommunicationObject client, Func<Task> action)
    {
        try
        {
            await action();
        }
        finally
        {
            CloseOrAbortServiceChannel(client);
        }
    }

    private static void CloseOrAbortServiceChannel(ICommunicationObject communicationObject)
    {
        if (communicationObject == null || communicationObject.State == CommunicationState.Closed)
            return;

        if (communicationObject.State == CommunicationState.Faulted)
        {
            communicationObject.Abort();
            Logger.Debug("The communication object is aborted after being in a faulted state.");
        }
        else
            communicationObject.Close();
    }
}

```

* To use it for synchronous calls:

```csharp
var client = InitializeClient();
client.CallAndClose(() =>
{
    var myresult = client.DoAction(myParameter);
});

```

* To use it for asynchronous calls:

```csharp
var client = InitializeClient();
await client.CallAndCloseAsync(async () => await client.DoActionAsync(myParameter));
```

Make sure that the client is not used again.  It must be used only inside the CallAndClose or CallAndCloseAsync methods.

