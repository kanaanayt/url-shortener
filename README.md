# URL Shortener Project - Commit History Analysis

This document provides an annotated timeline of the project's development through git commits, highlighting key changes and additions.

## Initial Project Setup
*Commit: dae03294e343585e2123f8773b6b3a9491b78ce1 to e3927309691e0eee8218a907925b9d370ff2402f*

In this first major commit, a basic ASP.NET Core API project was created with the standard Weather Forecast example.

### Project Structure
- Created a new .NET 8.0 web API solution `url.sln`
- Set up the API project with standard dependencies

### API Project Configuration
The `Api.csproj` file was created with the following configuration:

```csharp
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.11" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.6.2" />
  </ItemGroup>

</Project>
```

**Key Notes:**
- Using .NET 8.0 as the target framework
- Enabling nullable reference types for better null safety
- Enabling implicit usings to reduce boilerplate
- Adding OpenAPI/Swagger packages for API documentation

### API Endpoints
The standard Weather Forecast endpoint was created in `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapGet("/weatherforecast", () =>
{
    var forecast =  Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast")
.WithOpenApi();

app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

**Key Notes:**
- Using the minimal API approach (no controllers)
- Configuring Swagger for development environment
- Creating a simple weather forecast endpoint
- Using C# records for the data model

### HTTP Testing File
An HTTP test file was added to easily test the API:

```http
@Api_HostAddress = http://localhost:5056

GET {{Api_HostAddress}}/weatherforecast/
Accept: application/json

###
```

### Launch Settings
The project was configured with standard development launch settings:
- HTTP on port 5056
- HTTPS on port 7249
- Development environment settings

## CI Pipeline Setup
*Commit: e3927309691e0eee8218a907925b9d370ff2402f to 123dd402e8cc85b1767388e610ab90e159754187*

This commit added a GitHub Actions workflow for continuous integration.

### GitHub Actions Workflow

```yaml
# This workflow will build a .NET project
# For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-net

name: API

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: 8.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build --no-restore
    - name: Test
      run: dotnet test --no-build --verbosity normal
```

**Key Notes:**
- Triggers on pushes or pull requests to the main branch
- Runs on latest Ubuntu image
- Configures .NET 8.0 runtime
- Performs standard .NET workflow: restore, build, test

## Azure Deployment Setup
*Commit: 123dd402e8cc85b1767388e610ab90e159754187 to 9dcf1e8e4e01617cffff6b2fb076c006c0dabb77*

This commit added infrastructure as code (IaC) with Bicep templates and a GitHub Actions workflow for Azure deployment.

### Azure Deployment Workflow

```yaml
name: Azure Deploy

on:
  push:
    branches:
     - main
    paths:
     - infrastructure/** 
  pull_request:
    paths:
     - infrastructure/** 
  workflow_dispatch:

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment: Development
    steps:
     - uses: actions/checkout@v4

     - name: Azure login
       uses: azure/login@v2.1.1
       with:
         client-id: ${{ secrets.AZURE_CLIENT_ID }}
         tenant-id: ${{ secrets.AZURE_TENANT_ID }}
         subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
     
     - uses: Azure/CLI@v2
       with:
         inlineScript: |
           #!/bin/bash
           az group create --name ${{ vars.RESOURCE_GROUP_NAME }} --location ${{ vars.RESOURCE_GROUP_LOCATION }}
           echo "Azure resource group created"

    - name: Deploy
      uses: azure/arm-deploy@v2
      with:
        resourceGroupName: ${{ vars.RESOURCE_GROUP_NAME }}
        template: ./infrastructure/main.bicep
```

**Key Notes:**
- Triggers on pushes to main that modify infrastructure files
- Triggers on PRs that modify infrastructure files
- Can be manually triggered via workflow_dispatch
- Uses GitHub environment secrets for Azure authentication
- Creates Azure resource group if it doesn't exist
- Deploys Bicep templates

### Main Bicep Template
The main Bicep template orchestrates the deployment:

```bicep
param location string = resourceGroup().location
var uniqueId = uniqueString(resourceGroup().id)

module apiService 'modules/compute/appservice.bicep' = {
    name: 'apiDeployment'
    params: {
        appName: 'api-${uniqueId}'
        appServicePlanName: 'plan-api-${uniqueId}'
        location: location
    }
}
```

**Key Notes:**
- Uses resource group's location by default
- Generates a unique identifier based on resource group ID
- Calls the App Service module with generated unique names

### App Service Bicep Module
The App Service module defines the web app infrastructure:

```bicep
param appServicePlanName string 
param appName string 
param location string = resourceGroup().location

resource appServicePlan 'Microsoft.Web/serverfarms@2024-04-01' = {
    kind: 'linux'
    location: location
    name: appServicePlanName
    properties: {
        reserved: true
    }
    sku: {
        name: 'B1'
    }
}

resource webApp 'Microsoft.Web/sites@2024-04-01' = {
    name: appName
    location: location
    properties: {
        serverFarmId: appServicePlan.id
        httpsOnly: true
        siteConfig: {
            linuxFxVersion: 'DOTNETCORE|8.0'
        }
    }
}

resource webAppConfig 'Microsoft.Web/sites/config@2024-04-01' = {
    parent: webApp
    name: 'web'
    properties: {
        scmType: 'GitHub'
    }
}

output appServiceId string = webApp.id
```

**Key Notes:**
- Creates a Linux App Service Plan with B1 (Basic) SKU
- Creates a web app configured for .NET Core 8.0
- Enforces HTTPS only
- Configures GitHub as the SCM provider
- Outputs the App Service ID for reference

### README Updates
A README file was added with manual deployment commands:

```markdown
```bash
az group create --name urlshortener-dev --location westeurope

az deployment group create --resource-group urlshortener-dev --template-file infrastructure/main.bicep
```

**Key Notes:**
- Add two environment variables in GitHub Settings > Environments > Development
- These are RESOURCE_GROUP_NAME and RESOURCE_GROUP_LOCATION
- Now, Settings > Secrets and variables > Actions 
- Copy Azure subscription ID and paste that in Repository variables
- AZURE_SUBSCRIPTION_ID
