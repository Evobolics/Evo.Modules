# Evo.Modules
Allows a developer to create a dependency chain of modules and loads them in the proper order into memory.

# Introduction

This article desicusses the concepts related around building a host and how to extend the hosting model to support dynamically discoverable inter-dependent modules.  

Before extending the hosting concept further, it is important to understand what is currently present to prevent re-inventing the wheel and how to re-use what we have versus creating a brand new hosting model.

# Background

## What is a Host?

A host is an object that contains a collection of inter-dependent application resources with the goal of centrally managing its resources to provide a graceful application startup and shutdown process.  

Examples of resouces that hosts contain and manage incldue:

Configuration
Dependency injection (DI)
Logging

In addition to these basic services, .NET Core 3 hosts and later support the abilty to host services that that implement the [IHostedService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice?view=dotnet-plat-ext-5.0) interface.

## 2.0 Web and Host



## 3.0 ASP.NET Core rebuilt to work with Generic Host, instead of a Separate Web Host.



## Deprecation of the IHostingEnvronment &amp; IApplicationLifetime (Feb 14, 2019)

For more information please see issue [Deprecate and replace IHostingEnvronment &amp; IApplicationLifetime
](https://github.com/dotnet/extensions/issues/966).

- Microsoft.AspNetCore.Hosting.IHostingEnvironment
- Microsoft.AspNetCore.Hosting.IApplicationLifetime
- Microsoft.Extensions.Hosting.IHostingEnvironment
- Microsoft.Extensions.Hosting.IApplicationLifetime

```csharp

namespace Microsoft.Extensions.Hosting
{
    public interface IHostEnvironment
    {
        string EnvironmentName { get; set; }
        string ApplicationName { get; set; }
        string ContentRootPath { get; set; }
        IFileProvider ContentRootFileProvider { get; set; }
    }

    public interface IHostApplicationLifetime
    {
        CancellationToken ApplicationStarted { get; }
        CancellationToken ApplicationStopping { get; }
        CancellationToken ApplicationStopped { get; }

        void StopApplication();
    }

    public static class HostEnvironmentExtensions
    {
        public static bool IsDevelopment(this IHostEnvironment environment);
        public static bool IsStaging(this IHostEnvironment environment);
        public static bool IsProduction(this IHostEnvironment environment);
    }
}

```

# .NET 5 General Program Startup

There is a lot going on beyond what is visible to the developer when an application is created.  Developers live in a world today where they need to reference a relatively large number of packges when they create solutions.  Let's take a look what is going on under the hood when a worker service is being created.  The following code is what the Program.cs class contents look like when the application is first created in visual studio.

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace WorkerService1
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureServices((hostContext, services) =>
                {
                    services.AddHostedService<Worker>();
                });
    }
}

```

Notice there are two general parts.  The first is the ```Main``` method that is used to build and run the application.  The second part, is the ```CreateHostBuilder```.  This part of the application creates the builder class that can be used to configure the application prior to being built and ran.  This two part process seperaets the process of build and running the application from the process of configuring the application.  This is important, as it allows for applications to connected and participate with design time tools.

Lets examine the CreateHostBuilder method, and then circle back to the Main method where we can dicuss how the application is started.

## Creating the HostBuilder

The first goal is to create a host builder and configure the application.  To create a host builder, the standard way is to call Host.CreateDefaultBuilder().  This method is contained in the static [Host.cs](https://github.com/dotnet/extensions/blob/494e2c53cd0ea075ba3783748d62c66bc4a353e2/src/Hosting/Hosting/src/Host.cs) class.  

```csharp
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.EventLog;

namespace Microsoft.Extensions.Hosting
{
    
    public static class Host
    {
        
        public static IHostBuilder CreateDefaultBuilder() =>
            CreateDefaultBuilder(args: null);

        
        public static IHostBuilder CreateDefaultBuilder(string[] args)
        {
            var builder = new HostBuilder();

            builder.UseContentRoot(Directory.GetCurrentDirectory());
            builder.ConfigureHostConfiguration(config =>
            {
                config.AddEnvironmentVariables(prefix: "DOTNET_");
                if (args != null)
                {
                    config.AddCommandLine(args);
                }
            });

            builder.ConfigureAppConfiguration((hostingContext, config) =>
            {
                var env = hostingContext.HostingEnvironment;

                config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                      .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);

                if (env.IsDevelopment() && !string.IsNullOrEmpty(env.ApplicationName))
                {
                    var appAssembly = Assembly.Load(new AssemblyName(env.ApplicationName));
                    if (appAssembly != null)
                    {
                        config.AddUserSecrets(appAssembly, optional: true);
                    }
                }

                config.AddEnvironmentVariables();

                if (args != null)
                {
                    config.AddCommandLine(args);
                }
            })
            .ConfigureLogging((hostingContext, logging) =>
            {
                var isWindows = RuntimeInformation.IsOSPlatform(OSPlatform.Windows);

                // IMPORTANT: This needs to be added *before* configuration is loaded, this lets
                // the defaults be overridden by the configuration.
                if (isWindows)
                {
                    // Default the EventLogLoggerProvider to warning or above
                    logging.AddFilter<EventLogLoggerProvider>(level => level >= LogLevel.Warning);
                }

                logging.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
                logging.AddConsole();
                logging.AddDebug();
                logging.AddEventSourceLogger();

                if (isWindows)
                {
                    // Add the EventLogLoggerProvider on windows machines
                    logging.AddEventLog();
                }
            })
            .UseDefaultServiceProvider((context, options) =>
            {
                var isDevelopment = context.HostingEnvironment.IsDevelopment();
                options.ValidateScopes = isDevelopment;
                options.ValidateOnBuild = isDevelopment;
            });

            return builder;
        }
    }
}
```

A lot of work is done in this class.  Instead of the HostBuilder containing all of these default settings, the extension method provides them.  

Stepping through the code, the first thing the CreateDefaultBuilder method does is create an instance of the [HostBuilder](https://github.com/dotnet/extensions/blob/494e2c53cd0ea075ba3783748d62c66bc4a353e2/src/Hosting/Hosting/src/HostBuilder.cs) class.  When the ```HostBuilder.Build()``` method is called, the builder will proceed through five build stages:

- BuildHostConfiguration();
- CreateHostingEnvironment();
- CreateHostBuilderContext();
- BuildAppConfiguration();
- CreateServiceProvider();

Each stage provides some work to build the application and expose extension points that allow the developer, and in the case above, the ```CreateDefaultBuilder``` method, chances to configure the host. Later on in this article a more in depth review of what is happening within the HostBuilder will be provided, but for now we can summarize each step as follows:

The **BuildHostConfiguration** method creates the hosts configuration.  There are in fact two configurations that are built out during the course of building out a host, each created using a [ConfigurationBuilder](https://github.com/dotnet/runtime/blob/master/src/libraries/Microsoft.Extensions.Configuration/src/ConfigurationBuilder.cs).  The first is host configuration contains any settings needed to determine how the host should be built.  The second is the application configuration which is generally what developers access when requesting a copy of the configuration.

>*Note a lot of the extension libraries have been moved from the [extensions repo](https://github.com/dotnet/extensions) to the [dotnet runtime repo](https://github.com/dotnet/runtime) Old .NET Core 3.1 branches still remain in the extensions repo for historical reasons.  

The **CreateHostingEnvironment** method 

The **CreateHostBuilderContext**

The **BuildAppConfiguration**

The **CreateServiceProvider**

# References

* [Introducing IHostLifetime and untangling the Generic Host startup interactions](https://andrewlock.net/controlling-ihostedservice-execution-order-in-aspnetcore-3/)
* [Background tasks with hosted services in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-5.0&tabs=visual-studio#ihostedservice-interface)
