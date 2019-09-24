# Unit Conversion Service Workshop
Let’s see how F# language features make it simple to create a unit conversion service and deploy it to Azure Functions.

## Module 0: Background

### Who made this Workshop?
Brett Rowberry, who works at ExxonMobil

### When was this workshop first given?
Friday September 27th, 2019 at Open F# in San Francisco

### How long does this workshop take?
110 minutes

### What prerequisites will I need?
- An Azure account
- git (to clone this repository)
- Visual Studio Code
- Ionide extension
- Azure Functions extension
- Node.js 8.5+ (may be optional)
- .NET Core SDK 2.1
- Azure Functions Core Tools

### Why Azure Functions?

#### Why serverless?
- Potentially less expensive
- Potentially a simpler programming model

#### Why Azure Functions and not AWS Lambda or other serverless offerings?
- I use Azure at work

### What are the programming/deployment models supported by Azure Functions?
- [Script](https://docs.microsoft.com/en-gb/azure/azure-functions/functions-reference-fsharp)
- Compiled (what we'll use in this workshop)
- Container

## Module 1: C# Azure Function
There isn't an official F# template at the moment, so we'll start with a C# tutorial.

1. Open the `module1` directory in Visual Studio Code
2. Navigate to [Create your first function using Visual Studio Code](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-vs-code?WT.mc_id=devto-blog-aapowell)
3. Here are the main sections that we will go over together:
  1. Prerequisites  
    I think the easiest way is to:
    1. Get Visual Studio Code
    2. Install the `Azure Functions` extension
    3. Allow the extension to help you install other prerequisites
  2. Create your Functions project with a function (Create a folder called `1. CSharp`)
  3. Run the function locally and call it
  4. Publish the project to Azure 
    - Use the basic publish option (not advanced)
    - Name your app `module1<yourname>`
    - Use the Azure region closest to you. We'll use `West US` region since we're in San Francisco.
  5. Call the deployed API

## Module 2: F# Azure Function
1. Open the `module2` directory in Visual Studio Code
2. Create the C# project again, this time using the Azure Functions extension GUI
  - Change the function name to `HttpTriggerFSharp`
3. Navigate to [Azure Functions With F#](https://dev.to/azure/azure-functions-with-f-2l3c)
  1. Copy the code to the source file and change the extension from `.cs` to `.fs` (Ionide might look really upset at the file for a while, don't worry!)
  2. Change the extension of the project file from `.csproj` to `.fsproj`
  3. In the `.fsproj` file below the first `<ItemGroup>` section paste

  ``` XML
  <ItemGroup>
    <Compile Include="HttpTriggerFSharp.fs" />
  </ItemGroup>
  ```
  4. `POST`s aren't very fun to test. Let's change the function to a `GET` that uses query parameters like in Module 1.
    - Paste over the code with

``` F#
namespace Company.Function

open Microsoft.Azure.WebJobs
open Microsoft.Azure.WebJobs.Extensions.Http
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.Logging
open Microsoft.AspNetCore.Mvc
open System

module HttpTrigger =
    [<FunctionName("HttpTrigger")>]
    let Run([<HttpTrigger(AuthorizationLevel.Anonymous, "GET", Route = null)>] 
            req: HttpRequest,
            log: ILogger) 
            = 

        let name =
            try
                let maybeName = req.Query.["name"].[0]
                if String.IsNullOrWhiteSpace maybeName 
                then ValueNone
                else ValueSome maybeName
            with
                | :? IndexOutOfRangeException -> ValueNone

        match name with
        | ValueSome n -> 
            log.LogInformation(sprintf "name was '%s'" n)
            OkObjectResult(n) :> ActionResult
        | ValueNone -> 
            log.LogInformation("no name was passed")
            BadRequestObjectResult("Include a 'name' as a query string.") :> ActionResult
```
5. Run the function locally and call it
6. Publish the project to Azure
    - Use the advanced publish option using the GUI or Command Palette
    - Name your app `module2<yourname>`
    - Accept all other defaults
    - Choose Windows instead of Linux
      - [Streaming Logs "can't be used with an app running on Linux in a Consumption plan"](https://docs.microsoft.com/en-us/azure/azure-functions/functions-monitoring#streaming-logs)
7. There will be a prompt to stream logs, accept it
8. Call your app, inspect the logs
7. Navigate to https://portal.azure.com
8. Select your Function App
9. Disable and reenable the app
10. Run a test

### Module 3: Unit Conversion Service
1. Open the `module3` directory in Visual Studio Code
2. Create the same project as in module 2
  - Change the function name to `UnitConversionAPI`
  - This time we'll use route parameters instead of query parameters
  - Here's the code:
``` F#
namespace Company.Function

open System
open Microsoft.AspNetCore.Http
open Microsoft.Azure.WebJobs
open Microsoft.Azure.WebJobs.Extensions.Http
open Microsoft.Extensions.Logging

module Length =
    [<FunctionName("Length")>]
    let Run([<HttpTrigger(AuthorizationLevel.Anonymous, "GET", Route = "Length/{source}/{target}/{input}")>] 
            req: HttpRequest,
            source: string,
            target: string,
            input: string,
            log: ILogger) 
            = 
        let inputs = String.Join("|", source, target, input)
        log.LogInformation(sprintf "Inputs: '%s'" inputs)
        inputs 
```
3. Run the function locally and call it
4. Now that we have a working app, let's implement the conversion logic. Add a file above the existing file named `Length.fs`

``` F#
namespace UnitConversion

open System.Collections.Generic
open System.Linq
open System

type ConversionResult =
| NameNotFound of unitName : string * validUnits: string
| NamesNotFound of unitName : string * otherUnitName : string * validUnits: string
| Success of float

module Length =
    let private lengthsAndFactors =
        let fsharpDict =
            dict [
                "meter", 1.0
                "millimeter", 1e-3
                "kilometer", 1e3 ]
        Dictionary<string, float>(fsharpDict)

    let private tryGetUnitFactor name =
        let found, conversionFactor = lengthsAndFactors.TryGetValue name
        if found
        then Some conversionFactor
        else None

    let private lengths = 
        let lengths = lengthsAndFactors.Keys.ToArray()
        String.Join(", ", lengths)

    let convert source target input =
        match (tryGetUnitFactor source, tryGetUnitFactor target)  with
        | None, Some _ -> NameNotFound (source, lengths)
        | Some _, None -> NameNotFound (target, lengths)
        | None, None -> NamesNotFound (source, target, lengths)
        | Some s, Some t -> 
            input * s / t |> Success
```
5. Change your functions file to be:
``` F#
namespace Company.Function

open Microsoft.Azure.WebJobs
open Microsoft.Azure.WebJobs.Extensions.Http
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.Logging
open Microsoft.AspNetCore.Mvc
open System
open UnitConversion

module LengthAPI =
    open UnitConversion.Length
    [<FunctionName("LengthAPI")>]
    let Run([<HttpTrigger(AuthorizationLevel.Anonymous, "GET", Route = "length/{source}/{target}/{input}")>] 
            req: HttpRequest,
            source: string,
            target: string,
            input: float,
            log: ILogger) 
            = 
        let inputs = String.Join("|", source, target, input)
        log.LogInformation(sprintf "Inputs: '%s'" inputs)
        
        let result = convert source target input
        match result with
        | NameNotFound (s, lengths) ->
            let message = (sprintf "Length unit '%s' not found. Try %s." s lengths)
            log.LogError message

            message
            |> NotFoundObjectResult
            :> ActionResult
        | NamesNotFound (s, t, lengths) -> 
            let message = (sprintf "Length units '%s' and '%s' not found. Try %s." s t lengths)
            log.LogError message

            message
            |> NotFoundObjectResult
            :> ActionResult
        | Success r -> 
            log.LogInformation (sprintf "Conversion result: %f" r)

            OkObjectResult r 
            :> ActionResult
```

## More resources
[F# in 10 Minutes or Less | Precompiled Azure Functions V2 in Visual Studio Code](https://www.youtube.com/watch?v=SSBc5ucHq2E)
- [Using F# to write serverless Azure functions](https://blogs.msdn.microsoft.com/uk_faculty_connection/2017/03/24/using-f-to-write-serverless-azure-functions/) a little old
- [Work with Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local)
- [HTTP routing](https://docs.microsoft.com/en-us/sandbox/functions-recipes/routes?tabs=fsharp)
- [Precompiled Azure Functions in F#](https://mikhail.io/2017/12/precompiled-azure-functions-in-fsharp/)
- [Available templates for dotnet new](https://github.com/dotnet/templating/wiki/Available-templates-for-dotnet-new) - maybe you can make your own!
- [Giraffe.AzureFunctions](https://github.com/giraffe-fsharp/Giraffe.AzureFunctions)
- [Azure F#unctions](https://skillsmatter.com/skillscasts/11347-azure-f-sharpunctions)
- [DurableFunctions.FSharp](https://github.com/mikhailshilkov/DurableFunctions.FSharp)
- [Secure an Azure Function App with Azure Active Directory](http://www.mattruma.com/secure-an-azure-function-app-with-azure-active-directory/)
- [Tic-Tac-Toe with F#, Azure Functions, HATEOAS and Property-Based Tests](https://mikhail.io/2018/01/tictactoe-with-fsharp-azurefunctions-hateoas-and-property-based-testing/)
- [Introducing Saturn on Functions](https://medium.com/lambda-factory/introducing-saturn-on-functions-fa2338d431de)
- [fantomas-ui](https://gitlab.com/jindraivanek/fantomas-ui)
- [F# Yoga Class Booking System](https://github.com/Dzoukr/Yobo)
- [Azure Durable Functions in F#](https://mikhail.io/2018/02/azure-durable-functions-in-fsharp/)
- [azure-functions-durable-extension/samples/fsharp](https://github.com/Azure/azure-functions-durable-extension/tree/master/samples/fsharp)
- [A Fairy Tale of F# and Durable Functions](https://mikhail.io/2018/12/fairy-tale-of-fsharp-and-durable-functions/)
- [Creating custom bindings for Azure Functions](https://channel9.msdn.com/Shows/On-NET/Creating-custom-bindings-for-Azure-Functions)
- [azure-functions-fsharp-examples](https://github.com/mikhailshilkov/azure-functions-fsharp-examples)
