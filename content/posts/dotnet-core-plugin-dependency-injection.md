---
title: ".NET Core plugin dependency injection"
date: 2020-07-07T12:35:16+03:00
draft: false
---

# Introduction

You might run into a case where you would like to extend the application without updating the application core. Plugins could be one solution for the issue. It's fairly simple to implement a plugin architecture in .NET Core. Microsoft has documented the process of adding plugin support for your application in [https://docs.microsoft.com/en-us/dotnet/core/tutorials/creating-app-with-plugin-support](https://docs.microsoft.com/en-us/dotnet/core/tutorials/creating-app-with-plugin-support).

It's a good quick start to plugin architecture and gets you quite far. However, it does not explain how to add support for dependency injection from host process to the plugin process. Plugins own internal dependencies could be created on plugin initialization and injected to the executing code but I prefer to have the .NET Core dependency injection system handle these dependencies as well. Plugin configuration is a good example of an object I'd prefer to be initialized and injected by the .NET Core DI.

# Example case

Imagine we're developing a software where we receive messages from an external system and would like to deliver those to various other systems. We could implement the event handler as an Azure Function Appyo and add various integrations to other systems.

![Overview of the imaginary system](/images/dotnet-core-plugin-dependency-injection/overview.png)

Overview of the imaginary system

In the example code I've implemented the event handler as a .NET Core background service. The full example code is available at LINK

We'll use the .NET Core documentation plugin application example as the starting point for the system. In the example the project structure is following:

```
AppWithPlugin
PluginBase
HelloPlugin
```

In our case we'll be implementing something similar with slightly different names:

```
AppWithPlugin --> NetCorePluginDI
PluginBase --> NetCorePluginDI.PluginBase
HelloPlugin --> NetCorePluginDI.Integrations.*
```

Most of the code will stay the same.

As the plugins will be integrations, I've named the IPlugin interface from the original example to IIntegration. The rest of the interface stays mostly the same:

```csharp
public interface IIntegration
{
    string Name { get; }
    string Description { get; }
    Task ExecuteAsync(string message, CancellationToken cancellationToken);
}
```

This interface will be implemented by all of the plugins (Blob storage, HTTP and DB plugin). Below is a simplified version of the blob storage plugin.

```csharp
public class BlobStorageArchiveIntegration : IIntegration
{
    private readonly ILogger<BlobStorageArchiveIntegration> _logger;

    public BlobStorageArchiveIntegration(ILogger<BlobStorageArchiveIntegration> logger)
    {
        _logger = logger;
    }

    public string Name { get; } = "Blob Storage Archive";
    public string Description { get; } = "An integration to archive messages to Blob Storage";
    public Task ExecuteAsync(string message, CancellationToken cancellationToken)
    {
        // Let's pretend we're archiving to Blob Storage
        _logger.LogInformation($"Archiving message [{message}] to Blob Storage");

        return Task.CompletedTask;
    }
}
```

It will receive a message and "store" it to the blob storage (for demonstration purposes we'll just log the message).

# The problem

The above example with the original plugin example will work but there's one major issue: All of the plugins need some sort of configuration. Blob storage and DB integrations need a connection string, HTTP integration needs an URL to send the message to.

You could approach this by initializing the settings in the plugin by reading the settings from a file, injecting and using the IConfiguration interface or through some other method. However it would require you to define the settings initialization logic for each integration separately.

```csharp
public class BlobStorageArchiveIntegration : IIntegration
{
    private readonly ILogger<BlobStorageArchiveIntegration> _logger;

    public BlobStorageArchiveIntegration(ILogger<BlobStorageArchiveIntegration> logger)
    {
        _logger = logger;
    }

    public string Name { get; } = "Blob Storage Archive";
    public string Description { get; } = "An integration to archive messages to Blob Storage";

		public Task InitializeAsync()
		{
				// Read settings from somewhere and store them locally
				// ...
		}

    public Task ExecuteAsync(string message, CancellationToken cancellationToken)
    {
        // Let's pretend we're archiving to Blob Storage
        _logger.LogInformation($"Archiving message [{message}] to Blob Storage");

        return Task.CompletedTask;
    }
}
```

I'd prefer not having to specify initialization logic on each integration. Additionally this is pretty much a solved problem by the .NET Core DI, we could just use it instead.

# The solution

Let's define a class that we'll inject to the integration.

```csharp
public class HttpIntegrationSettings
{
    public string Url { get; set; }
}
```

Let's use the settings in the integration.

```csharp
public class HttpIntegration : IIntegration
{
    private readonly ILogger<HttpIntegration> _logger;
    private readonly HttpIntegrationSettings _settings;

    public HttpIntegration(ILogger<HttpIntegration> logger, HttpIntegrationSettings settings)
    {
        _logger = logger;
        _settings = settings;
    }

    public string Name { get; } = "HTTP";
    public string Description { get; } = "An integration to send messages to a REST endpoint";
    public Task ExecuteAsync(string message, CancellationToken cancellationToken)
    {
        // Let's pretend we're sending messages to another service
        _logger.LogInformation($"Sending message [{message}] to [{_settings.Url}]");

        return Task.CompletedTask;
    }
}
```

Now the question is, how will we inform the DI framework about the HttpIntegrationSettings class and that it should load the settings from the IConfiguration providers?

We can achieve that by changing the module loading code. Let's start by defining an interface that all injected settings classes will be implementing.

```csharp
public interface ISettings
{
}
```

Let's just define an empty interface. It's just to declare the type is injectable as an interface. We could also use a naming convention such as if type name ends in Settings, we'll inject it as settings. I prefer to be explicit.

Once we've added the ISetting interface and implemented it in HttpIntegrationSettings, we can implement the additional plugin code to instantiate and inject settings instances to the service provider.

```csharp
private static void RegisterIntegrationsFromAssembly(IServiceCollection services, IConfiguration configuration, Assembly assembly)
{
    foreach (var type in assembly.GetTypes())
    {
        // Register all classes that implement the IIntegration interface
        if (typeof(IIntegration).IsAssignableFrom(type))
        {
            // Add as a singleton as the Worker is a singleton and we'll only have one
            // instance. If this would be a Controller or something else with clearly defined
            // scope that is not the lifetime of the application, use AddScoped.
            services.AddSingleton(typeof(IIntegration), type);
        }

        // Register all classes that implement the ISettings interface
        if (typeof(ISettings).IsAssignableFrom(type))
        {
            var settings = Activator.CreateInstance(type);
            // appsettings.json or some other configuration provider should contain
            // a key with the same name as the type
            // e.g. "HttpIntegrationSettings": { ... }
            if (!configuration.GetSection(type.Name).Exists())
            {
                // If it does not contain the key, throw an error
                throw new ArgumentException($"Configuration does not contain key [{type.Name}]");
            }
            configuration.Bind(type.Name, settings);

            // Settings can be singleton as we'll only ever read it
            services.AddSingleton(type, settings);
        }
    }
}
```

We'll register the integration as previously. For all interfaces implementing the ISettings interface we'll create the instance for the type. Then we'll ensure that the type name exists as a key in the settings (e.g. appsettings.json, environment variables or in another configuration provider). If the key exists, let's bind the object to the instance and register the settings to the service provider.

This way we'll able utilize the existing infrastructure to load settings for the plugins.

Let's run the application:

```jsx
Loading integrations from: \Path\To\NetCorePluginDI\NetCorePluginDI.Integrations.BlobStorageArchive\bin\Debug\netstandard2.0\NetCorePluginDI.Integrations.BlobStorageArchive.dll
Loading integrations from: \Path\TNetCorePluginDI\NetCorePluginDI.Integrations.Http\bin\Debug\netstandard2.0\NetCorePluginDI.Integrations.Http.dll
info: NetCorePluginDI.Integrations.BlobStorageArchive.BlobStorageArchiveIntegration[0]
      Archiving message [Worker running at: 6.7.2020 21.51.14 +03:00] to Blob Storage
info: NetCorePluginDI.Integrations.Http.HttpIntegration[0]
      Sending message [Worker running at: 6.7.2020 21.51.14 +03:00] to [https://localhost:3000/api/message]
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: \Path\To\\NetCorePluginDI\NetCorePluginDI
info: NetCorePluginDI.Integrations.BlobStorageArchive.BlobStorageArchiveIntegration[0]
      Archiving message [Worker running at: 6.7.2020 21.51.15 +03:00] to Blob Storage
info: NetCorePluginDI.Integrations.Http.HttpIntegration[0]
      Sending message [Worker running at: 6.7.2020 21.51.15 +03:00] to [https://localhost:3000/api/message]
```

As you can see, we'll load integrations from two DLLs, the Blob Storage and HTTP integration DLLs. The system receives a message every second and sends it to the integrations. In the log output you can see HttpIntegration logging the endpoint url that was defined in the project appsettings.json (https://localhost:3000/api/message).

This can be extended even further. If we have a type we'd like to inject to the plugin we can implement another interface. I'll call it IInjectedDependency. It's an empty interface. All types that implement this interface will be registered to the service provider.

```csharp
public interface IInjectedDependency
{
}

public class Service : IInjectedDependency
{
		public void DoWork()
		{
				// ...
		}
}
```

This way we can fully utilize the .NET Core DI framework within the plugins.
