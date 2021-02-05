# Evo.Modules
Allows a developer to create a dependency chain of modules and loads them in the proper order into memory.

# Program Startup

```csharp
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
```

# Microsoft Hosting Implementation

## Build

### 
