# Serilog.Extensions.Hosting [![Build status](https://ci.appveyor.com/api/projects/status/ue4s7htjwj88fulh?svg=true)](https://ci.appveyor.com/project/serilog/serilog-extensions-hosting) [![NuGet Version](http://img.shields.io/nuget/v/Serilog.Extensions.Hosting.svg?style=flat)](https://www.nuget.org/packages/Serilog.Extensions.Hosting/) 

Serilog logging for _Microsoft.Extensions.Hosting_. This package routes framework log messages through Serilog, so you can get information about the framework's internal operations written to the same Serilog sinks as your application events.

### Instructions

**First**, install the _Serilog.Extensions.Hosting_ [NuGet package](https://www.nuget.org/packages/Serilog.Extensions.Hosting) into your app. You will need a way to view the log messages - _Serilog.Sinks.Console_ writes these to the console; there are [many more sinks available](https://www.nuget.org/packages?q=Tags%3A%22serilog%22) on NuGet.

```powershell
Install-Package Serilog.Extensions.Hosting -DependencyVersion Highest
Install-Package Serilog.Sinks.Console
```

**Next**, in your application's _Program.cs_ file, configure Serilog first.  A `try`/`catch` block will ensure any configuration issues are appropriately logged:

```csharp
public class Program
{
    public static int Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .CreateLogger();

        try
        {
            Log.Information("Starting host");
            BuildHost(args).Run();
            return 0;
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Host terminated unexpectedly");
            return 1;
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }
```

**Then**, add `UseSerilog()` to the host builder in `BuildHost()`.

```csharp    
    public static IHost BuildHost(string[] args) =>
        new HostBuilder()
            .ConfigureServices(services => services.AddSingleton<IHostedService, PrintTimeService>())
            .UseSerilog() // <- Add this line
            .Build();
}
```

**Finally**, clean up by removing the remaining `"Logging"` section from _appsettings.json_ files (this can be replaced with [Serilog configuration](https://github.com/serilog/serilog-settings-configuration) as shown in [this example](https://github.com/serilog/serilog-extensions-hosting/blob/dev/samples/SimpleServiceSample/Program.cs), if required)

That's it! You will see log output like:

```
[22:10:39 INF] Getting the motors running...
[22:10:39 INF] The current time is: 12/05/2018 10:10:39 +00:00
```

A more complete example, showing _appsettings.json_ configuration, can be found in [the sample project here](https://github.com/serilog/serilog-hostinh/tree/dev/samples/SimpleServiceSample).

### Using the package

With _Serilog.Extensions.Hosting_ installed and configured, you can write log messages directly through Serilog or any `ILogger` interface injected by .NET. All loggers will use the same underlying implementation, levels, and destinations.

**Tip:** change the minimum level for `Microsoft` to `Warning` 

### Inline initialization

You can alternatively configure Serilog using a delegate as shown below:

```csharp
    // dotnet add package Serilog.Settings.Configuration
    .UseSerilog((hostingContext, loggerConfiguration) => loggerConfiguration
	.ReadFrom.Configuration(hostingContext.Configuration)
	.Enrich.FromLogContext()
	.WriteTo.Console())
```

This has the advantage of making the `hostingContext`'s `Configuration` object available for configuration of the logger, but at the expense of recording `Exception`s raised earlier in program startup.

If this method is used, `Log.Logger` is assigned implicitly, and closed when the app is shut down.

### Writing to the Azure Diagnostics Log Stream

The Azure Diagnostic Log Stream ships events from any files in the `D:\home\LogFiles\` folder. To enable this for your app, first install the _Serilog.Sinks.File_ package:

```powershell
Install-Package Serilog.Sinks.File
```

Then add a file sink to your `LoggerConfiguration`, taking care to set the `shared` and `flushToDiskInterval` parameters:

```csharp
    public static int Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
            .Enrich.FromLogContext()
            .WriteTo.Console()
	    // Add this line:
	    .WriteTo.File(
	    	@"D:\home\LogFiles\Application\myapp.txt",
		fileSizeLimitBytes: 1_000_000,
		rollOnFileSizeLimit: true,
		shared: true,
		flushToDiskInterval: TimeSpan.FromSeconds(1))
            .CreateLogger();
```
