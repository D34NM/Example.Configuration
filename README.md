# Configuration API in .NET Core

The configuration provides an easy way, without recompiling or changing the code, to change the behavior of the application. This way we can keep, for example environment specific settings, in the configuration of the application.

The configuration support in .NET Core is quite extensive and supports a multiple way to configure the application. Compared to previous versions of .NET framework, the configuration is now part of the `Microsoft.Extensions.Configuration` NuGet package. This package is by default included in .NET Core projects templates. The configuration is defined as a collection of the `key-value` pairs. The example and short explanation are bellow:

```csharp
IReadOnlyDictionary<string, string> Configurations = new Dictionary<string, string>
{
    { "AppConfiguration:OutputDirectory", "Path/To/The/Directory" },
    { "AppConfiguration:ConnectionString", "SomeConnectionStringForTheDatabase" }
};
```

The example above is defining 2 configurations for the application in code (will move away from this later). The syntax is easy to understand `AppConfiguration:OutputDirectory = "Path/To/The/Directory"`, as it follows the hierarchy:

```text
AppConfiguration
|--OutputDirectory = Path/To/The/Directory
|--ConnectionString = SomeConnectionStringForTheDatabase
```

This means that configuration can have sections that provide more context about what part of the application will be using the configuration. Quick example:

```text
ApiConfiguration:LogLevel = "Info"
StorageConfiguration:ConnectionString = "SomeConnectionString"
ThirdPartyClient:BaseUrl = "www.third-party.com"
```

The character `:` is used as a separator between sections (not the case for Environment Variables and Azure Key Vault). Support for the sections reduces the risk of having the problem of defining "strange" names for the keys, to prevent the duplicate unique keys. As long as the key is unique within its section, there is no problem:

```text
ApiConfiguration:LogLevel = "Info"
StorageConfiguration:LogLevel = "Debug"
```

The configuration key is case insensitive, that means `LogLevel == loglevel`.

## Configuration providers

There are multiple ways to define the configuration in your projects. JSON files, INI files, Environment variables, etc. And if none of this are not working out for the particular use-case, you can always define your own.

This is where the Configuration Providers come into the play. There are multiple of providers coming along with the .NET Core (or their respective packages):

1. [File configuration provider](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-3.1#file-configuration-provider)
2. [Memory configuration provider](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-3.1#memory-configuration-provider)
3. [Environment variables configuration provider](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-3.1#evcp)
4. And [others](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-3.1#cp)

The order of the configuration providers matters and it will make a difference in what values will be loaded.

On the [MSDN](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-3.1#custom-configuration-provider) you can find information on how to define a custom configuration provider.

## The appsettings.json and appsettings.{env}.json

The common pattern when defining configuration in most .NET Core applications is to use `JSON` files. This also comes by default with some of the templates within the framework (`appsettings.json` should come to mind). An example of the configuration being defined in the `appsettings.json`:

```json
{
    "AppConfiguration": {
        "OutputDirectory": "Path/To/The/Directory",
        "ConnectionString": "SomeConnectionStringForTheDatabase"
    }
}
```

To make it more explicit and keep the configuration specific for different environments separated, the convention is to have them in `appsettings.{env}.json`. The `env` indicates all different environments will be running in (example: `Production`, `Test`, `Development`).

## IConfiguration

This is the main contract that you will be using to access the configuration. The configuration that is loaded (using any of the providers) will be "flattened". This means it will be mapped to the structure from the beginning of the document, `Section:ConfigurationKey`. Example of how we can access the configuration value:

```csharp
public MyClass(IConfiguration configuration)
{
    var logLevel = configuration.GetValue<string>("AppConfiguration:LogLevel");
    var outputDirectory = configuration["AppConfiguration:OutputDirectory"];
}
```

## IConfiguration and strongly typed configuration

So it is possible to get the configuration using the `IConfiguration` API. Tho, this is not the best way to fetch the values from our application configuration. Simply as it requires introduction of the magic strings, we need to know the configuration keys, etc.

The better approach is to define strongly typed objects. They can represent the entire configuration or they can represent a segment of it. The example bellow shows this in detail:

```json
{
    "Example": {
        "Configuration1": "Configuration 1",
        "Configuration2": 1,
        "Configuration3": true
    }
}
```

```csharp
public class ExampleOptions
{
    public string Configuration1 { get; set; }
    public int Configuration2 { get; set; }
    public bool Configuration3 { get; set; }
}
```

```csharp
public MyConstructor(IConfiguration configuration)
{
    this.configuration = configuration;
}

// Somewhere in the code :)
var options = new ExampleOptions();
configuration.Bind("Example", options);

if (options.Configuration3) 
{
    // some code goes here
}
```

This is a first step, towards more cleaner way, but it shows how it is possible to bind the configuration values to the instance of the strongly typed object. The naming of the properties need to match the naming of the configuration keys. Tho, this can be fixed if by some reason this is not the case. To solve this, we can use the standard JSON serialization attributes to indicate the names of the properties and their mapping to the strongly typed object.

## IOptions pattern and dependency injection

The `IOptions` pattern allows binding of the configuration to the strongly typed objects. This is similar to what we saw in the previous section. One of the bigger differences is that this pattern works seamlessly with the Dependency Injection inside of the .NET Core.

```json
{
    "Example": {
        "Configuration1": "Configuration 1",
        "Configuration2": 1,
        "Configuration3": true
    }
}
```

```csharp
public class ExampleOptions
{
    public string Configuration1 { get; set; }
    public int Configuration2 { get; set; }
    public bool Configuration3 { get; set; }
}
```

```csharp
public MyConstructor(IOptions<ExampleOptions> options)
{
    this.configuration = options.Value;
}

// Somewhere in the code :)
if (configuration.Configuration3)
{
    // some code goes here
}
```

To make this work, we also need, when creating a DI container, configure the binding. This is achieved by the following example:

```csharp
// services is instance of IServiceCollection
services.Configure<ExampleOptions>(configuration.GetSection("Example"));
```

## IOptionsSnapshot

In cases that underlying values in the file changes, till application is run again we are not aware of this change. This can be configured with to offer the fetching of the new values on changes made to the file.

To support this, we need to replace the dependency on the `IOptions<T>` towards the `IOptionsSnapshot<T>`. There is a simple reason for this. The `IOptions<T>` is registered as a singleton within DI container, while `IOptionsSnapshot<T>` is registered as scoped dependencies.

This change should be applied really carefully tho. Especially in cases where this can possibly lead towards some inconsistent states between changes. Or if it will drastically change how our code behaves.

## IOptionsMonitor

In case that we need to have configuration reloading reflected in the code during runtime and we need to depend on it in the object that is registered as a `singleton` in the DI container, `IOptionsSnapshot<T>` will not work.

For this purpose, we can use the `IOptionsMonitor<T>`. It is registered as a `singleton` in the DI container as well, tho it will provide a way to reflect the changes to the configuration in runtime. Example bellow shows this:

```json
{
    "Example": {
        "Configuration1": "Configuration 1",
        "Configuration2": 1,
        "Configuration3": true
    }
}
```

```csharp
public class ExampleOptions
{
    public string Configuration1 { get; set; }
    public int Configuration2 { get; set; }
    public bool Configuration3 { get; set; }
}
```

```csharp
private readonly IOptionsMonitor<ExampleOptions> configuration;
public MyConstructor(IOptionsMonitor<ExampleOptions> options)
{
    this.configuration = options;
}

// Somewhere in the code :)
if (configuration.CurrentValue.Configuration3)
{
    // some code goes here
}
```

## Named options

In case that we need to reuse the same strongly typed object for multiple different configurations, instead of creating the multiple copies of the same object, we can use named options. This only works with `IOptionsSnapshot<T>` or `IOptionsMonitor<T>`.

```csharp
// services is instance of IServiceCollection
services.Configure<ExampleOptions>("Example1", configuration.GetSection("Example"));
services.Configure<ExampleOptions>("Example2", configuration.GetSection("Example2"));

// Somewhere in the code
public MyConstructor(IOptionsMonitor<ExampleOptions> options)
{
    this.configuration1 = options.Get("Example1");
}
```

## Validating configurations

In case that we make sure that our configuration is valid before using it, we can apply validation to our strongly typed models when loading the configuration. This is a good practice if we have something that could lead towards inconsistent and incorrect behavior of the application.

```csharp
using System.ComponentModel.DataAnnotations;

public class ExampleOptions
{
    [Required]
    public string Configuration1 { get; set; }
    public int Configuration2 { get; set; }
    public bool Configuration3 { get; set; }
}
```

```csharp
// services is an instance of the IServiceCollection
services.AddOptions<ExampleOptions>(
        .Bind(configuration.GetSection("Example"))
        .ValidateDataAnnotations();
)
```

This will still fail only when we access the value in runtime. If application should not even start if the configuration is invalid, we need to be a bit creative. .NET Core at the moment ([Github issue #459](https://github.com/dotnet/extensions/issues/459)). There are some examples online how we can solve this issue: using a background service, a custom extension method that access all the properties in the instance, etc. I will try to provide some examples in the solutions.

In case you need some specific validation that are not maybe covered by using annotations, you can also use the following:

```csharp
// services is an instance of the IServiceCollection
services.AddOptions<ExampleOptions>(
        .Bind(configuration.GetSection("Example"))
        .Validate(options => // options is of type ExampleOptions
        {
            if (options.Configuration1 != "Some specific value")
            {
                return false;
            }

            return true;
        });
```

There is an overload of the `Validate` method that also allows you to specify the error message, in case that validation fails. Chaining of the `Validate` is also possible and you can split the validation (together maybe with proper error messages) in more granular and specific functions. In case of the validation errors, application won't even start. So this could be one of the ways to go around of the issue when using the annotations.

### IConfigureOptions<T>

If there is a need to construct configuration based, for example, on some third party requests (or some more complex computation), you can reach out for `IConfigureOptions<T>` interface. It allows you to, like regular service, use DI to inject dependencies and other fun stuff,

```csharp
public class ConfigureExampleOptionsOptions : IConfigureOptions<ExampleOptions>
{
    private readonly IExternalServiceClient _client;

    public ConfigureMySettingsOptions(IExternalServiceClient client)
    {
        _client = client;
    }

    public void Configure(ExampleOptions options)
    {
        options.Configuration1 = _client.DoSomeWork();
    }
}
```

Trying to achieve similar thing using the `Configure()` method leads to some "funny" looking code. This is a nice way to indicate there is some "complexity" about this particular configuration. And encapsulate it within single place. One nice thing, it "stacks" with standard `Configure()`. So you can load some values from `appsettings.json` and then do some complex calculations for others:

```csharp
var builder = WebApplication.CreateBuilder(args);
var configuration = builder.Configuration;

builder.Services.Configure<ExampleOptions>(configuration.GetSection("ExampleOptions")); 

builder.Services.AddSingleton<IConfigureOptions<ExampleOptions>, ConfigureExampleOptionsOptions>();

builderServices.AddSingleton<IExternalServiceClient>();
```

### IValidateOptions

It is also possible to move the validation code into its own place, to make the code cleaner and more maintainable. This is where the `IValidateOptions<T>` is good for.

```csharp
class ExampleOptionsValidator : IValidateOptions<ExampleOptions>
{
    public ValidateOptionsResult Validate(string name, ExampleOptions options)
    {
        if (options.Configuration1 != "Some specific value")
        {
            return ValidateOptionsResult.Fail("Well, you can't do that!!!");
        }

        return ValidateOptionsResult.Success;
    }
}
```

Nice thing about this approach is that it also supports DI, so you can inject some other services (or maybe some other configurations using `IOptions` pattern) to perform more complex validations.

```csharp
// services is instance of IServiceCollection
services.Configure<ExampleOptions>(configuration.GetSection("Example"));
services.AddSingleton<IValidateOptions<>, ExampleOptions>();
```

This will also prevent the application from starting in case that there are any validation errors. So one more way to pass around the data annotations issue with eager validations. The `name` parameter in the `IValidateOptions<T>.Validate(string name, T options)` is used for validating of the named options. This will provide a name of the options that can be used for specific validations.

### IOptions vs. IOptionsSnapshot vs. IOptionsMonitor

|                     | Can be used in singletons      | Supports Reloading | Named Options |
| ------------------- | ------------------------------ | ------------------ | ------------- |
| IOptions            | Yes                            | No                 | No            |
| IOptionsSnapshot    | No                             | Yes                | Yes           |
| IOptionsMonitor     | Yes                            | Yes                | Yes           |

## Unit testing

There are two ways we can solve this, but lets first start with using a factory method that comes with `Microsoft.Extensions.Options`.

```csharp
[Fact]
public void Test01()
{
    var exampleOptions = new ExampleOptions { Configuration1 = "Some string", Configuration2 = 1, Configuration3 = false };
    IOptions<ExampleOptions> options = Options.Create(exampleOptions);
    // pass it to the class being tested as a parameter
}
```

The second option here is to use mocking. This depends on your framework of choice to define the mocks, so example can defer so will be omitted in this case.
