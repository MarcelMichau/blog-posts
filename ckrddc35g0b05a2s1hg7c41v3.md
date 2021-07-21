## Taming Startup.cs in ASP.NET Core

Photo by [Safar Safarov](https://unsplash.com/@codestorm?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/code?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

---

## Overview
There are plenty of ASP.NET Core projects out on the internet. All of them have one thing in common. The `Startup.cs` file. There's also another thing that a lot of these `Startup.cs` files have in common - they are **HUGE**.

## The Problem
For those unfamiliar with `Startup.cs`, this is the class that wires up _pretty much everything_ in ASP.NET Core. It's where the request pipeline, dependency injection container, configuration & others get configured. The official docs can be found [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup). The issue with being responsible for wiring up _pretty much everything_ is that this class tends to grow quite large as more things need to be configured. This problem is compounded with plenty of documentation & tutorials suggesting that, to wire up [_insert feature here_], all you need to do is drop [_insert code snippet here_] into `Startup.cs` and you're off to the races. And the file just keeps on growing...

This causes a few things:
- The `Startup` class becomes difficult to navigate because there's a lot happening in it
- The `Startup` class violates the Single Responsibility Principle (SRP) because it no longer has a single reason to change
- It hurts maintainability because there's a high chance of merge conflicts if multiple developers make changes to the `Startup` class simultaneously

## The Problem in Practice
Let's take the following `Startup.cs` template as a starting point, taken from the .NET Core Web API template (comments removed for brevity):

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseHttpsRedirection();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```
Awesome, clean & simple. For now.

Now let's add our first feature. Any application with reasonable complexity is going to need to store some data somewhere. So, let's add Entity Framework Core & wire it up with SQL Server. As per the docs, we add the following to `ConfigureServices`:

```csharp
services.AddDbContext<SchoolContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("SchoolContext")));
```

We also want to be good citizens & document our API so that it can be used by others. So let's use [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle.AspNetCore) to add some Open API (Swagger) documentation to our API. So we add the following to `ConfigureServices`:

```csharp
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My Awesome API", Version = "v1" });
});
```

And the following to `Configure`, inside the `env.IsDevelopment()` check:

```csharp
app.UseSwagger();
app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "My Awesome API v1"));
```

We've also written a couple of custom classes & interfaces which we need to add to the `ServiceCollection` for Dependency Injection, so let's add them to `ConfigureServices`:

```csharp
services.AddScoped<INotificationService, EmailNotificationService>();
services.AddScoped<INotificationService, SmsNotificationService>();
services.AddScoped<INotificationService, PushNotificationService>();
```

That's enough features for today. Not too many, but enough to start off with. Now our `Startup` class looks like this:

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();

        services.AddDbContext<SchoolContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("SchoolContext")));

        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", new OpenApiInfo { Title = "My Awesome API", Version = "v1" });
        });

        services.AddScoped<INotificationService, EmailNotificationService>();
        services.AddScoped<INotificationService, SmsNotificationService>();
        services.AddScoped<INotificationService, PushNotificationService>();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
            app.UseSwagger();
            app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "My Awesome API v1"));
        }

        app.UseHttpsRedirection();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

It's not too hairy quite yet, but we've only added three features. Imagine what this will look like once we add authentication, authorization, validation, exception handling, logging, caching, customizing Swagger, etc. It can get out of hand for a reasonably-sized application.

## The Alternative
The following approach aims to address these issues by refactoring our `Startup.cs` file so that we can all sleep a little better at night, and we all know that sleep is important.

Let's try to separate some of the `Startup` class' concerns by refactoring some of them to separate classes. We can use some of the same tricks used by the ASP.NET Core team to accomplish this. For example, if we look at the following line:

```csharp
services.AddControllers();
```

What that's doing is configuring all the necessary bits & pieces so that we can use Controllers in an ASP.NET Core Web API, including stuff like Authorization, CORS, Data Annotations, ApiExplorer, etc. But it's all just one line of code. Neat.

That is also an _extension method_ that operates on `IServiceCollection`. So let's try and write some of our own extension methods which work with `IServiceCollection`, starting with the database:

```csharp
internal static class DatabaseServiceCollectionExtensions
{
    public static IServiceCollection AddDatabaseConfiguration(this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<SchoolContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("SchoolContext")));

        return services;
    }
}
```
Here we have a static class named `DatabaseServiceCollectionExtensions` because it contains extension methods which operate on `IServiceCollection`.

There is one static method, `AddDatabaseConfiguration` which takes the `IServiceCollection` & `IConfiguration` as parameters and returns an `IServiceCollection`. This method then calls the same code we had in `Startup` to configure Entity Framework Core, and then returns the updated `IServiceCollection`.

Then we can use it in our `Startup` class inside `ConfigureServices` like so:

```csharp
services.AddDatabaseConfiguration(Configuration);
```

The `Configuration` argument is optional for these extension methods, it's just there for the case where the extension method needs something from config in order to perform it's logic, as is the case with the `AddDatabaseConfiguration` method.

We can then continue this trend for our own application services:

```csharp
internal static class ApplicationServicesServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServicesConfiguration(this IServiceCollection services)
    {
        services.AddScoped<INotificationService, EmailNotificationService>();
        services.AddScoped<INotificationService, SmsNotificationService>();
        services.AddScoped<INotificationService, PushNotificationService>();    

        return services;
    }
}
```

And use it in `Startup`:

```csharp
services.AddApplicationServicesConfiguration();
```

As for Swagger, we needed to make changes to both `ConfigureServices` and `Configure` in order to wire it up. Therefore,we'll need two extension methods, one which operates on `IServiceCollection` as before, and one which operates on `IApplicationBuilder` for the logic in the `Configure` method:

```csharp
internal static class SwaggerExtensions
{
    public static IServiceCollection AddSwaggerConfiguration(this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", new OpenApiInfo { Title = "My Awesome API", Version = "v1" });
        });

        return services;
    }

    public static IApplicationBuilder UseSwaggerConfiguration(this IApplicationBuilder app)
    {
        app.UseSwagger();

        return app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "My Awesome API v1"));
    }
}
```

Then we just use them in `Startup` in their respective methods. In `ConfigureServices`:

```csharp
services.AddSwaggerConfiguration();
```

And in `Configure`:
```csharp
app.UseSwaggerConfiguration();
```

After all this, our `Startup` class looks like this:
```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddDatabaseConfiguration(Configuration);
        services.AddApplicationServicesConfiguration();
        services.AddSwaggerConfiguration();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
            app.UseSwaggerConfiguration();
        }

        app.UseHttpsRedirection();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

That's a bit better. 😁

## Conclusion

This approach has a few benefits:
- It keeps the `Startup` class focused by moving configuration logic out to separate classes
- It prevents the `Startup` class from growing uncontrollably
- It maintains a better separation of concerns such that, when something changes in the configuration of Swagger, for example, that change is isolated to the class which deals with Swagger configuration
- It improves maintainability because it's easier to find & navigate to the class which deals with a specific feature rather than scrolling through the entirety of the `Startup` class looking for the specific line which needs to be changed

I hope this has proved useful to someone & that it helps you make your applications' code slightly simpler. 😀