# Exercise #2

## Goal

Explore Externalized Configuration by working with Spring Cloud Config Server

## Expected Results

Create an instance of Spring Cloud Services Configuration Server and bind our API application to that instance of the Configuration Server.

## Introduction

In this exercise we explore how Configuration Server pulls configuration from a backend repository.  Also observe how we utilize Steeltoe to connect to Configuration Server and manipulate how we retrieve configuration.

1. Create a directory for our new API with the following command:  `mkdir bootcamp-webapi`

2. Navigate to the newly created directory using the following command: `cd bootcamp-webapi`

3. Use the Dotnet CLI to scaffold a basic Web API application with the following command: `dotnet new webapi`.  This will create a new application with name bootcamp-webapi.  **Note the project will take the name of the folder that the command is run from unless given a specific name**

4. Navigate to the project file (the file with a .csproj extension) and edit it to add the following nuget packages.  Nuget is the .NET package manager for managing external libraries.  Adding these lines tells Nuget the package and version to pull into the project.  More information about Nuget can be found [here](https://www.nuget.org/)

    ```xml
    <PackageReference Include="NSwag.AspNetCore" Version="11.19.2" />
    <PackageReference Include="Pivotal.Extensions.Configuration.ConfigServerCore" Version="2.1.1" />
    ```

5. In the Program.cs class add the following using statement and edit the CreateWebHostBuilder method in the following way.  The using statement allows us to use types of a given namespace without fully qualifying a given type [see](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-directive) for information on the c# using statement.

    ```c#
    using Pivotal.Extensions.Configuration.ConfigServer;
    ```

    ```c#
    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseCloudFoundryHosting()
            .ConfigureAppConfiguration(b => b.AddConfigServer(new LoggerFactory().AddConsole(LogLevel.Trace)))
            .UseStartup<Startup>();
    ```

    **Take note of the UseCloudFoundryHosting and AddConfigServer methods which are extensions that allow us to listen on a configured port and adds spring cloud config server as a configuration source respectively.  For further information see the Steeltoe Configuration [docs](https://steeltoe.io/docs/steeltoe-configuration/)**

6. Navigate to the Startup class and set the following using statements:

    ```c#
    using Pivotal.Extensions.Configuration.ConfigServer;
    using NJsonSchema;
    using NSwag.AspNetCore;
    ```

7. The Startup.ConfigureServices method is responsible for defining the services the app uses.  In the ConfigureServices method use an extension method to add swagger and the Config Server provider to the DI Container with the following lines of code.  For a in depth look at dependency injection in ASP.NET Core see the following [article](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.1)

    ```c#
    services.AddConfiguration(Configuration);
    services.AddSwagger();
    ```

    ***The workshop was built and tested against .NET Core and ASP.NET CORE 2.1.  If you have a newer version of the SDK installed it may be necessary to modify the compatibility version of the application as follows:***

        ```c#
        // change to following line of code
        services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
        //to
        services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
        ```

8. The Configure method is used to specify how the app responds to specific HTTP requests.  In the Configure method add swagger to the middleware pipeline by adding the following code snippet just before the `app.UseMvc();` line.  This configures the pipeline to serve the Swagger specification based on our application.  For information on application start up in ASP.NET Core [see](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup?view=aspnetcore-2.1)

    ```c#
    app.UseSwaggerUi3WithApiExplorer(settings =>
        {
            settings.GeneratorSettings.DefaultPropertyNameHandling = 
                PropertyNameHandling.CamelCase;
            settings.PostProcess = document => 
            {
                document.Info.Version = "v1";
                document.Info.Title = "Bootcamp API";
                document.Info.Description = "A simple ASP.NET Core web API";
                document.Schemes.Clear();
                document.Schemes.Add(NSwag.SwaggerSchema.Https);
            };
            settings.SwaggerUiRoute = "";
        });
    ```

9. A controller is used to define and group a set of actions which are methods that handle incoming requests.  For an in depth view of ASP.NET Core MVC [see](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview?view=aspnetcore-2.1) and for a Controller specific discussion [see](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/actions?view=aspnetcore-2.1).  We previously added the Spring Cloud Config Server to our service container in step 5 which allows us access to configuration specified in the server.  In the following code snippet we create a controller class, inject the configuration into the controller and locate specific configuration keys in an action method.  In the controllers folder create a new class and name it ProductsController.cs and then paste the following contents into the file:

    ```c#
    using System;
    using System.Collections.Generic;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Extensions.Configuration;

    namespace bootcamp_webapi.Controllers
    {
        [Route("api/[controller]")]
        [ApiController]
        public class ProductsController : Controller
        {
            private readonly IConfigurationRoot _config;
            public ProductsController(IConfigurationRoot config)
            {
                _config = config;
            }

            // GET api/products
            [HttpGet]
            public IEnumerable<string> Get()
            {
                Console.WriteLine($"product1 is {_config["product1"]}");
                Console.WriteLine($"product2 is {_config["product2"]}");
                return new[] {$"{_config["product1"]}", $"{_config["product2"]}"};
            }
        }
    }
    ```

10. In the root directory navigate to the appsettings.json file and add an entry for spring and spring cloud config like the below snippet.  These settings tell Eureka to register our service instance with the Eureka Server

    ```json
    "spring": {
      "application": {
        "name": "api"
      },
      "cloud": {
        "config": {
          "name": "api",
          "env": "dev",
          "validateCertificates" : false
        }
      }
    }
    ```

11. In the application root create a file and name it config.json and edit it in the following way.  The settings in this file will tell our instance of Spring Cloud Config that we will be connecting to a git repository at the location specified in the uri field.

    ```json
    {
        "git": {
        "uri": "https://github.com/tezizzm/cloud-native-net-configs"
        }
    }
    ```

12. Run the following command to create an instance of Spring Cloud Config Server with settings from config.json **note: service name and type may be different depending on platform/operator configuration**

    ```bat
    cf create-service p-config-server standard myConfigServer -c .\config.json
    ```

13. You are ready to now “push” your application.  Create a file at the root of your application name it manifest.yml and edit it to match the snippet below.  The settings in this file instruct the cloud foundry cli how to stage and deploy your application. **Note due to formatting issues simply copying the below manifest file may produce errors due to the nature of yaml formatting.  Use the CloudFoundry extension recommend in exercise 1 to assist in the correct formatting**

    ```yml
    ---
    applications:
    - name: dotnet-core-api
    random-route: true
    buildpack: https://github.com/cloudfoundry/dotnet-core-buildpack
    instances: 1
    memory: 256M
    env:
    ASPNETCORE_ENVIRONMENT: development
    services:
    - myConfigServer
    ```

14. Run the cf push command to build, stage and run your application on PCF.  Ensure you are in the same directory as your manifest file and type `cf push`.

15. Once the `cf push` command has completed navigate to the given url and you should see the Swagger page.  If you navigate to the api/products path you should see a list of products that are pulled from configuration.  To confirm the configuration have a look at the configured git repo in the config.json file.