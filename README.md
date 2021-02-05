# Evo.Modules
Allows a developer to create a dependency chain of modules and loads them in the proper order into memory.

# Program Startup

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

# Microsoft Hosting Implementation

## Build

### 
