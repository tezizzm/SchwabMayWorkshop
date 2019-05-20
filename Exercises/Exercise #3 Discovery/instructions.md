# Exercise #3

## Goal

Explore service registration and discovery practices.

## Expected Results

Register existing microservice with a service registry. Create a client application so that it discovers the microservice and uses it.

## Introduction

This exercise helps us understand how to register our microservices with the Spring Cloud Services Registry, and also discover those services at runtime.

1. Return back to your `bootcamp-webapi` project root and find the project file.  We will add the following nuget package:

    ```xml
    <PackageReference Include="Pivotal.Discovery.ClientCore" Version="2.1.1" />
    ```

2. Navigate to the Startup class and set the following using statements:

    ```c#
    using Pivotal.Discovery.Client;
    ```

3. In the ConfigureServices method use an extension method to add the discovery client to the DI Container with the following line of code.  This extension methods adds the Discovery Services to the service container.

    ```c#
    services.AddDiscoveryClient(Configuration);
    ```

4. In the Configure method add the discovery client to the middleware pipeline by adding the following code snippet.  This code configures the pipeline for the discovery client.

    ```c#
    app.UseDiscoveryClient();
    ```

5. In the root directory navigate to the appsettings.json file and add an entry for eureka like the below snippet.  These settings tell Eureka to register our service instance with the Eureka Server

    ```json
    "eureka": {
      "client": {
          "shouldRegisterWithEureka": true,
          "shouldFetchRegistry": false,
          "validateCertificates" : false
      }
    }
    ```

6. Run the following command to create an instance of Service Discovery **note: service name and type may be different depending on platform/operator configuration**

    ```bat
    cf create-service p-service-registry standard myDiscoveryService
    ```

7. Navigate to the manifest.yml file and in the services section add an entry to bind the application to the newly created instance of the Service Discovery Service.

    ```yml
        - myDiscoveryService
    ```

8. We will now once again push the API application.  Run the `cf push` command to update the api.

9. Go "manage" the `Service Registry` instance from within Apps Manager. Notice our service is now listed!

We now change focus to a front end application that discovers our products API microservice.

1. Navigate to the workspace root.  Create a directory for our new UI with the following command:  `mkdir bootcamp-store`

2. Navigate to the newly created directory using the following command: `cd bootcamp-store`

3. Use the Dotnet CLI to scaffold a basic MVC application with the following command: `dotnet new mvc`.  This will create a new application with the name bootcamp-store.

4. Navigate to the project file and edit it to add the following nuget packages:

    ```xml
    <PackageReference Include="Pivotal.Discovery.ClientCore" Version="2.1.1" />
    <PackageReference Include="Steeltoe.Discovery.ClientCore" Version="2.1.1" />
    <PackageReference Include="Steeltoe.Extensions.Configuration.CloudFoundryCore" Version="2.1.1" />
    ```

5. In the Program.cs class add the following using statement and edit the CreateWebHostBuilder method in the following way.  Notice the extension method AddCloudFoundry.  It is used to add the [VCAP variables](https://docs.run.pivotal.io/devguide/deploy-apps/environment-variable.html) to the configuration root.

    ```c#
    using Steeltoe.Extensions.Configuration.CloudFoundry;
    ```

    ```c#
    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseCloudFoundryHosting(5555)
            .AddCloudFoundry()
            .UseStartup<Startup>();
    ```

6. Navigate to the Startup class and set the following using statements:

    ```c#
    using Pivotal.Discovery.Client;
    ```

7. In the ConfigureServices method use an extension method to add the discovery client to the DI Container with the following line of code.

    ```c#
    services.AddDiscoveryClient(Configuration);
    ```

    The workshop was built and tested against .NET Core and ASP.NET CORE 2.1. If you have a newer version of the SDK installed it may be necessary to modify the compatibility version of the application as follows:

    ```c#
    // change to following line of code
    services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
    //to
    services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
    ```

8. In the Configure method add the following code to configure the discovery client middleware.

    ```c#
    app.UseDiscoveryClient();
    ```

9. Edit the HomeController.cs class to retrieve our products from the API. First add the appropriate using statements to bring in namespace references.  Then notice the `DiscoveryHttpClientHandler` property.  It maps our call to a discovered service instance and then completes the service request.  Once the request is complete we log the results and pass the data on to our view for display.  In this case the view is an MVC view, you can read more about views [here](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/overview?view=aspnetcore-2.1).  Once complete the file should look like the following:

    ```c#
        using System;
        using System.Diagnostics;
        using System.Net.Http;
        using System.Threading.Tasks;
        using Microsoft.AspNetCore.Mvc;
        using bootcamp_store.Models;
        using Steeltoe.Common.Discovery;

        namespace bootcamp_store.Controllers
        {
            public class HomeController : Controller
            {
                //private static HttpClient client = new HttpClient();
                DiscoveryHttpClientHandler handler;

                public HomeController(IDiscoveryClient client)
                {
                    handler = new DiscoveryHttpClientHandler(client);
                }

                public async Task<IActionResult> Index()
                {
                    var client = new HttpClient(handler, false);
                    var result = await client.GetAsync("https://dotnet-core-api/api/products");
                    var products = await result.Content.ReadAsAsync<string[]>();
                    ViewData["products"] = products;
                    foreach (var product in products)
                    {
                        Console.WriteLine(product);
                    }
                    return View();
                }

                [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
                public IActionResult Error()
                {
                    return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
                }
            }
        }
    ```

    **Take note of the url format (<https://dotnet-core-api/api/products>) of the external API call.**

10. Navigate to the views folder and edit the View file named Index.cshtml file to match the below snippet.  This file uses a mix of html and Razor syntax to iterate over and display the products returned from the Products API.  You can read about Razor Syntax [here](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/razor?view=aspnetcore-2.1)

    ```c#
    @{
        ViewData["Title"] = "Home Page";
    }

    <div class="jumbotron">
        <div class="container">
            <h1>Bootcamp Store</h1>
            <h2>Welcome to the Bootcamp Store please see a listing of products below</h2>
        </div>
    </div>
    <div class="container">
        <h2>Products</h2>
        @if (ViewData["Products"] is string[] products)
        {
            foreach (var product in products)
            {
                <div class="row">
                    <div class="col-xs-12">
                        <ul>
                            <li>@product</li>
                        </ul>
                    </div>
                </div>
            }
        }
    </div>
    ```

11. In the root directory navigate to the appsettings.json file and add an entry for eureka like follows.  Notice since we are consuming the service we do not register with Eureka.

    ```json
    "spring": {
      "application": {
        "name" : "ui"
      },
      "eureka": {
        "client": {
            "shouldRegisterWithEureka": false,
            "shouldFetchRegistry": true,
            "validateCertificates": false
        }
      }
    }
    ```

12. You are ready to now “push” your application.  Create a file at the root of your application name it manifest.yml and edit it as follows:  **Note due to formatting issues simply copying the below manifest file may produce errors due to the nature of yaml formatting.  Use the CloudFoundry extension recommend in exercise 1 to assist in the correct formatting**

    ```yml
    ---
    applications:
    - name: dotnet-core-ui
    buildpack: https://github.com/cloudfoundry/dotnet-core-buildpack
    random-route: true
    instances: 1
    memory: 256M
    services:
    - myDiscoveryService
    ```

13. Run the cf push command to build, stage and run your application on PCF.  Ensure you are in the same directory as your manifest file and type `cf push`.

14. Once the command has completed, navigate to the url to see the home page with products listed.