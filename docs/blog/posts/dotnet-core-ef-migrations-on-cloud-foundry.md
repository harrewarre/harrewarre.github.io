---
date: 2024-06-10  
slug: dotnet-ef-migrations-on-cloud-foundry  
tags:  
- dotnet core
- entity framework
- cloud foundry
title: Dotnet Core Entity Framework Migrations on Cloud Foundry
---
# Dotnet Core Entity Framework Migrations on Cloud Foundry

Run database migrations on Cloud Foundry using a task to ensure that migrations are only done once on deployment (and not during app start up!) for dotnet core applications. This post assumes you are deploying built applications and not raw source code.
<!-- more -->

Instead of depending on dotnet tooling (`dotnet-ef`), leverage the built-in migration capability of the DataContext. Interrupt the startup procedure when a special argument is passed in, instead of starting the web application, run the migrations and then exit.

Have a DbContext:
```c#
using Microsoft.EntityFrameworkCore;  
  
namespace MyApp.Secondary;  
  
public class MyDbContext : DbContext  
{  
    public MyDbContext(DbContextOptions<MyDbContext> options)  
        : base(options)  
    { }
    
    public DbSet<User> Users { get; set; }
}
```

In the solution that I was working on we had a factory that can create a DbContext when one is needed.
```c#
using Microsoft.EntityFrameworkCore;  
  
namespace MyApp.Secondary;  
  
public interface IMyDbContextFactory
{  
    MyDbContextFactory Create();  
}  
  
public class MyDbContextFactory(IConfiguration configuration) : IMyDbContextFactory  
{  
    public MyDbContextFactory Create()  
    {
	    var services = new VcapServices(configuration);  
        var connectionString = services.GetPostGresConnectionString("my-database-service");  
        var optionsBuilder = new DbContextOptionsBuilder<MyDbContext>();  
        optionsBuilder            
	        .UseNpgsql(connectionString)  
            .UseSnakeCaseNamingConvention();  
  
        return new MyDbContextFactory(optionsBuilder.Options);  
    }}
```

Enable migrations, this will work locally, but I had no clue on how to get this working through the Cloud Foundry buildpack, which inspired this post.

```c#
builder.Services.AddSingleton<IMyDbContextFactory, MyDbContextFactory>();
```

To get the migrations rolling we still use `dotnet ef` tooling locally.
```bash
dotnet ef migrations add MyMigration
```

The code (that lives in `Program.cs`) below will take the startup code down a different branch when the `--migrate` argument is passed to the application. Because all the services and configuration are registered we can request our factory from the initialized service collection and grab an instance of our `DbContext` instance. Using this context we can apply all pending migrations using the `context.Database.Migrate()` method.

```c#
// Service and Configuration registration.

var app = builder.Build();

if (args.FirstOrDefault(string.Empty) == "--migrate")  
{  
    var contextFactory = app.Services.GetService<IMyDbContextFactory>();
    if (contextFactory == null)  
    {
	    throw new Exception("ContextFactory could not be located!");    
    }
    
    Console.WriteLine("Applying database migrations...");  
    var context = contextFactory.Create();  
    context.Database.Migrate();  
    Console.WriteLine("Migrations applied successfully!");
    
    return;  
}

// More bootstrapping

app.Run();
```

During deployment on the platform we want to apply migrations when the app is deployed but not yet started. We can deploy our app with 0 instances and then run a task on the platform that applies the migrations by passing the `--migrate` argument.

```bash
cf push -i 0
```

After deployment but before startup (there are no instances running yet), run a Cloud Foundry task against the app. The app assembly is the result of a `dotnet publish` command that we run in our CI system, so this solution assumes you are deploying built applications, not source.

If you should take the app completely offline is an interesting topic all on its own. Depending on your migrations and what you are trying to accomplish you might be able to get away without downtime but for the sake of safety, this example accepts a very brief period of downtime as the migrations are being applied to ensure that migrations are applied in a single task, and we don't risk multiple app instances all trying to apply the migrations at the same time as they are starting up.

```bash
cf run-task my-app --command "\"./my-app-assembly --migrate\"" --name migrate-database --wait
```

When the task completes successfully and our database is in its desired state we can scale the app to the desired count.

```bash
cf scale my-app -i 2
```

Local migrations can be applied using the same setup:

```bash
dotnet run --project my-app-project.csproj -- --migrate
```

The `dotnet_core_buildpack` does not seem (I was unable to find anything related to additional `dotnet` tools) to be able to use tools like `dotnet ef`. Using the solution outlined in this post you can work around not having the tools and instead let the application apply the migrations.

You'll want to roll all this into some kind of CI/CD system because these manual steps are easy to mess up. See you in the next one!